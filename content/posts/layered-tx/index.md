---
title: "レイヤードアーキテクチャでトランザクションをエレガントに抽象化する方法2選"
date: 2023-12-16T12:14:44+09:00
---

この記事は技術書典で弊社から出した本『[SGE Go Tech Book Vol.04](https://creator.game.cyberagent.co.jp/?p=10211)』自分が書いた章を書き直したものです。

レイヤードアーキテクチャにおけるトランザクションの実装（主に抽象化）の方法について、2つの例を紹介します。

<!--more-->

本章のサンプルコードは次のURLから参照できます。

https://github.com/karamaru-alpha/tx-layer

## tl;dr

以下2つのパターンを紹介する。後者の方がかっこいい気がしている。

- ContextにTxオブジェクトを詰め伝搬するパターン
- Txオブジェクトを抽象化しRepositoryに引数で渡すパターン

## 実装パターン1 TxオブジェクトをContextに詰める

本節では、レイヤードアーキテクチャにおけるトランザクション処理の実装パターンとして「TxオブジェクトをContextに詰める」方法を紹介します。

<i>package構成</i>

```shell
$ tree ./context-pattern/
├── domain
│	├── repository
│	│	└── user.go
│	└── transaction
│		└── tx_manager.go
├── infra
│	└── mysql
│		├── repository
│		│	└── user.go
│		└── tx_manager.go
├── usecase
│	└── user.go
└── xcontext
└── xcontext.go
```

### Contextとは

GoにはContextという概念があります。
[パッケージの説明](https://pkg.go.dev/context#pkg-overview)にもある通り、Contextの主な用途は以下3つとされています。

- 処理の締め切りの伝達
- キャンセル信号の伝達
- リクエストスコープ値の伝達

> Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.

「TxオブジェクトをContextに詰める」というこの実装パターンは、Contextにおける「リクエストスコープ値の伝達」という働きを用いて、トランザクションを管理するTxオブジェクトを各層へ伝搬することで、データベースへの直接的な依存をせずにトランザクションを呼び出す方法になります。

Contextに詰められるリクエストスコープ値の代表例は認証情報などですが、1リクエストを通じてデータ不整合を起こさないようにするという意味で、Txオブジェクトをリクエストスコープ値として扱うのもContextの考え方から大きく外れないと解釈しています。

### 実装

実装を見ていきましょう。まずはトランザクションを呼び出すアプリケーション層です。

<i>context-pattern/usecase/user.go</i>
```go
type UserInteractor interface {
    UpdateName(ctx context.Context, userID, name string) error
}

type userInteractor struct {
    txManager      transaction.TxManager
    userRepository repository.UserRepository
}

func NewUserInteractor(
    txManager transaction.TxManager,
    userRepository repository.UserRepository,
) UserInteractor {
    return &userInteractor{
        txManager,
        userRepository,
    }
}

func (i *userInteractor) UpdateName(ctx context.Context, userID, name string) error {
    // NOTE: トランザクションを開始する
    if err := i.txManager.Transaction(ctx, func(ctx context.Context) error {
        user, err := i.userRepository.SelectByPK(ctx, userID)
        if err != nil {
            return err
        }
    
        user.Name = name
        if err := i.userRepository.Update(ctx, user); err != nil {
            return err
        }
    
        return nil
    }); err != nil {
        return err
    }

    return nil
}
```

`txManager.Transaction`の第2引数に指定する関数はトランザクション内で実行されます。
トランザクションの抽象を保持した`txManager`をDI（依存性の注入）することで、アプリケーション層がデータベースの関心を持たずトランザクションを呼び出せています。

次に、トランザクションの抽象を定義し、アプリケーション層とインフラストラクチャ層の中継役も担うドメイン層を見ていきましょう。こちらもデータベースの関心を持たずにトランザクションを表現できていますね。

<i>context-pattern/domain/transaction/tx_manager.go</i>
```go
type TxManager interface {
    Transaction(ctx context.Context, f func(context.Context) error) error
}
```


続いて、トランザクションを実装するインフラストラクチャ層を見ていきます。
MySQLのトランザクションを開始し、その際に得られるTxオブジェクトをContextに詰めてから、トランザクション関数の引数に渡された関数を実行しています。

(みやすさのため、MySQLとの疎通にはGoの標準パッケージdatabase/sqlの拡張である[sqlxパッケージ](https://github.com/jmoiron/sqlx)を用いています)

<i>context-pattern/infra/mysql/tx_manager.go</i>

```go
type txManager struct {
    db *sqlx.DB
}

func NewTransactionManager(db *sqlx.DB) transaction.TxManager {
    return &txManager{
        db,
    }
}

func (t *txManager) Transaction(ctx context.Context, f func(context.Context) error) error {
    tx, err := t.db.BeginTxx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        // panic -> rollback
        if p := recover(); p != nil {
            if err := tx.Rollback(); err != nil {
                slog.ErrorContext(ctx, "failed to MySQL Rollback")
            }
            panic(p)
        }
        // error -> rollback
        if err != nil {
            if e := tx.Rollback(); e != nil {
                slog.ErrorContext(ctx, "failed to MySQL Rollback")
            }
            return
        }

        // success -> commit
        if e := tx.Commit(); e != nil {
            slog.ErrorContext(ctx, "failed to MySQL Commit")
        }
    }()

    // NOTE: TxオブジェクトをContextにセットしてから引数の関数を実行する
    ctx = xcontext.WithValue[xcontext.Transaction](ctx, xcontext.Transaction{
        Tx: tx,
    })
    err = f(ctx)
    if err != nil {
        return err
    }
    return nil
}
```


Tipsですが、Contextにおける値の出し入れはファントム型（型パラメタによって型を出し分ける方法）を利用することで次のように書くことができます。

cf. https://hypirion.com/musings/spectral-contexts-in-go

<i>context-pattern/xcontext/xcontext.go</i>
```go
type Transaction struct {
    Tx *sqlx.Tx
}

type keyConstraint interface {
    Transaction
}

type key[T keyConstraint] struct{}

func WithValue[T keyConstraint](ctx context.Context, val T) context.Context {
    return context.WithValue(ctx, key[T]{}, val)
}

func Value[T keyConstraint](ctx context.Context) (T, bool) {
    val, ok := ctx.Value(key[T]{}).(T)
    return val, ok
}
```


最後に、MySQLに対してクエリを発行するインフラストラクチャ層のRepository実装です。
Contextの中にTxオブジェクトが存在するか確認し、存在すればトランザクション内でクエリを発行しています。

<i>context-pattern/infra/mysql/repository/user.go</i>

```go
type userRepository struct {
    db *sqlx.DB
}

func NewUserRepository(db *sqlx.DB) repository.UserRepository {
    return &userRepository{
        db,
    }
}

func (r *userRepository) Update(ctx context.Context, e *entity.User) error {
    db := r.getMysqlDB(ctx)
    
    if _, err := db.ExecContext(ctx, "UPDATE users SET name = ? WHERE user_id = ?", e.Name, e.UserID); err != nil {
        return err
    }
    return nil
}

// NOTE: ContextにTxオブジェクトが存在すればそれを返却し、存在しなければDIされたdbを返却する
func (r *userRepository) getMysqlDB(ctx context.Context) db {
    if transaction, ok := xcontext.Value[xcontext.Transaction](ctx); ok {
        return transaction.Tx
    }
    return r.db
}

// NOTE: トランザクション内外で共通のデータベース操作インターフェース
type db interface {
    ExecContext(ctx context.Context, query string, args ...any) (sql.Result, error)
}
```

以上が「TxオブジェクトをContextに詰める」パターンの実装でした。
TxオブジェクトをContext内に隠蔽しアプリケーション層とインフラストラクチャ層を疎結合を保つことで、レイヤードアーキテクチャの利点を崩さずにトランザクションを表現できています。

この方法のメリットは、各層でContextさえ受け取っていれば好きなタイミングでトランザクションの開始・Txオブジェクトの伝搬ができることです。middlewareでTxオブジェクトをContextに詰める処理を記述することで、すべてのエンドポイントでトランザクションを実現することも可能です。

一方のデメリットとして、Txオブジェクトの受け渡しがContext内部に隠蔽されていることによる見通しの悪さが挙げられます。
コードが複雑化するにつれ、ある関数がトランザクション内で呼ばれることを期待しているのかが表面上のコード/シグニチャからは判別しづらくなる可能性があります。

そこで、Txオブジェクトの受け渡しまで抽象化して各層で扱えるようにした実装を次節で紹介します。



## 実装パターン2 Txオブジェクトを抽象化・DIする

本節では、レイヤードアーキテクチャにおけるトランザクション処理の実装パターンとして「Txオブジェクトを抽象化しDIする」方法を紹介します。

また、この実装パターンと相性のいい、ReadOnlyなトランザクションとReadWriteなトランザクションの使い分けも同時に紹介しようと思います。

<i>package構成</i>
```shell
$ tree ./di-pattern/
├── domain
│	├── repository
│	│    └── user.go
│	└── transaction
│	     └── tx_manager.go
├── infra
│	└── mysql
│		├── repository
│		│    └── user.go
│		├── tx.go
│		└── tx_manager.go
└── usecase
└── user.go
```

まずはトランザクションの呼び出しを行うアプリケーション層から見ていきます。
ReadOnly/ReadWriteなトランザクションの発行が可能な`txManager`をDIすることで、データベースへの直接的な依存をせずにトランザクションを扱うことができています。


<i>di-pattern/usecase/user.go</i>
```go
type UserInteractor interface {
    GetUser(ctx context.Context, userID string) (*entity.User, error)
    UpdateName(ctx context.Context, userID, name string) error
}

type userInteractor struct {
    txManager      transaction.TxManager
    userRepository repository.UserRepository
}

func NewUserInteractor(
    txManager transaction.TxManager,
    userRepository repository.UserRepository,
) UserInteractor {
return &userInteractor{
        txManager,
        userRepository,
    }
}

func (i *userInteractor) GetUser(ctx context.Context, userID string) (*entity.User, error) {
    var user *entity.User
    
    // NOTE: ReadOnlyなトランザクションを開始する
    if err := i.txManager.ReadOnlyTransaction(ctx, func(ctx context.Context, tx transaction.ROTx) error {
        var err error
        user, err = i.userRepository.SelectByPK(ctx, tx, userID)
        if err != nil {
            return err
        }
        return nil
    }); err != nil {
        return nil, err
    }
    return user, nil
}

func (i *userInteractor) UpdateName(ctx context.Context, userID, name string) error {
    // NOTE: ReadWriteなトランザクションを開始する
    if err := i.txManager.ReadWriteTransaction(ctx, func(ctx context.Context, tx transaction.RWTx) error {
        user, err := i.userRepository.SelectByPK(ctx, tx, userID)
        if err != nil {
            return err
        }
        user.Name = name
        if err := i.userRepository.Update(ctx, tx, user); err != nil {
            return err
        }

        return nil
    }); err != nil {
        return err
    }
    return nil
}
```

次に、ReadOnly/ReadWriteなトランザクションと、それぞれに対応するTxオブジェクトの抽象を定義しているドメイン層を見ていきます。
ReadWriteなトランザクションの中でも、参照系の（ROTxを引数として受け取る）Repositoryを呼び出せるように、RWTxはROTxを満たすように設計しています。

<i>di-pattern/domain/transaction/tx_manager.go</i>

```go
// ReadOnlyなTxオブジェクトの抽象
type ROTx interface {
    ROTxImpl()
}

// ReadWriteなTxオブジェクトの抽象 (ROTxも満たす)
type RWTx interface {
    ROTx
    RWTxImpl()
}

type TxManager interface {
    ReadOnlyTransaction(ctx context.Context, f func(ctx context.Context, tx ROTx) error) error
    ReadWriteTransaction(ctx context.Context, f func(ctx context.Context, tx RWTx) error) error
}
```

Txオブジェクトとトランザクションの実装はインフラストラクチャ層にあります。

まずはTxオブジェクトの実装です。
ドメイン層で定義されたReadOnly/ReadWriteそれぞれのTxオブジェクトの抽象を実装する型（roTx/rwTx）と、抽象からそれらを逆引きできる関数（ExtractROTx/ExtractRWTx）を用意します。
ReadWriteなトランザクションの中でもROTxを引数として受け取るRepositoryを呼び出せるよう、ExtractROTxではroTxだけでなくrwTxも取り出せることに留意してください。

ちなみに、参考実装で使用しているMySQLにはReadOnlyなトランザクションが存在しないためTxオブジェクトの実体をそれぞれ同じ型（*sqlx.Tx）で表現しています。
Google CloudのNewSQLサービスであるSpannerなど、ReadOnlyなトランザクションが機能として存在する場合は、それに対応するTxオブジェクトをroTxにセットすることで効率的なトランザクションの使い分けを実現できます。

<i>di-pattern/infra/mysql/tx.go</i>

```go
// ドメイン層で定義されたReadOnlyなTxオブジェクトの抽象ROTxを実装する型
type roTx struct {
    // MysqlにはReadOnlyなTxオブジェクトが存在しないためrwTxと同じ*sqlx.Tx型を利用している
    *sqlx.Tx
}

// 引数の型を制限するための役割なので実装は空
func (t *roTx) ROTxImpl() {}

// ドメイン層で定義されたReadWriteなTxオブジェクトの抽象RWTxを実装する型
type rwTx struct {
    *sqlx.Tx
}

// 引数の型を制限するための役割なので実装は空
func (t *rwTx) ROTxImpl() {}
func (t *rwTx) RWTxImpl() {}

// rwTx/roTx共通のインターフェース
type ReadTx interface {
    GetContext(ctx context.Context, dest interface{}, query string, args ...interface{}) error
}

// ドメイン層で定義されたReadOnlyなTxオブジェクトの抽象ROTxから実体のTxオブジェクトを取り出す
func ExtractROTx(_tx transaction.ROTx) (ReadTx, error) {
    switch tx := _tx.(type) {
    case *roTx:
        return tx, nil
    case *rwTx:
        // NOTE: ReadWriteなトランザクションの中でも参照系の(ROTxを引数として受け取る)Repositoryを呼び出せるように、実体がrwTxの場合も許容する
        return tx, nil
    }
    return nil, errors.New("mysql ROTx is invalid")
}

// ドメイン層で定義されたReadWriteなTxオブジェクトの抽象RWTxから実体のTxオブジェクトを取り出す
func ExtractRWTx(_tx transaction.RWTx) (*rwTx, error) {
    tx, ok := _tx.(*rwTx)
    if !ok {
        return nil, errors.New("mysql RWTx is invalid")
    }
    return tx, nil
}
```

次にトランザクションの実装です。
MySQLのトランザクションを開始し、その際に得られるTxオブジェクトを上記で定義したroTx/rwTx型でラップし、それをトランザクション関数の引数で受け取った関数（usecase）に渡し、実行します。

<i>di-pattern/infra/mysql/tx_manager.go</i>
```go
type txManager struct {
    db *sqlx.DB
}

func NewTxManager(db *sqlx.DB) transaction.TxManager {
    return &txManager{db}
}

func (t *txManager) ReadWriteTransaction(ctx context.Context, f func(context.Context, transaction.RWTx) error) error {
    tx, err := t.db.BeginTxx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
        // panic -> rollback
        if p := recover(); p != nil {
            if err := tx.Rollback(); err != nil {
                slog.ErrorContext(ctx, "failed to MySQL Rollback")
            }
            panic(p)
        }
        // error -> rollback
        if err != nil {
            if e := tx.Rollback(); e != nil {
                slog.ErrorContext(ctx, "failed to MySQL Rollback")
            }
            return
        }
        // success -> commit
        if e := tx.Commit(); e != nil {
            slog.ErrorContext(ctx, "failed to MySQL Commit")
        }
    }()

    // NOTE: ReadWriteTransactionオブジェクトを関数に渡す
    err = f(ctx, &rwTx{tx})
    if err != nil {
        return err
    }
    return nil
}

func (t *txManager) ReadOnlyTransaction(ctx context.Context, f func(context.Context, transaction.ROTx) error) error {
    tx, err := t.db.BeginTxx(ctx, nil)
    if err != nil {
        return err
    }
    defer func() {
    // panic -> rollback
    if p := recover(); p != nil {
        if err := tx.Rollback(); err != nil {
            slog.ErrorContext(ctx, "failed to MySQL Rollback")
        }
        panic(p)
    }
    // error -> rollback
    if err != nil {
        if e := tx.Rollback(); e != nil {
            slog.ErrorContext(ctx, "failed to MySQL Rollback")
        }
        return
    }
    // success -> commit
    if e := tx.Commit(); e != nil {
        slog.ErrorContext(ctx, "failed to MySQL Commit")
    }
    }()

    // NOTE: ReadOnlyTransactionオブジェクトを関数に渡す
    err = f(ctx, &roTx{tx})
    if err != nil {
        return err
    }
    return nil
}
```

最後にRepositoryの実装です。
引数からTxオブジェクトを取り出し、トランザクション内でクエリを発行します。

<i>di-pattern/infra/mysql/repository/user.go</i>
```go
type userRepository struct {}

func NewUserRepository() repository.UserRepository {
    return &userRepository{}
}

type User struct {
    UserID string `db:"user_id"`
    Name   string `db:"name"`
}

func (u *User) toEntity() *entity.User {
    return &entity.User{
        UserID: u.UserID,
        Name:   u.Name,
    }
}

func (r *userRepository) SelectByPK(ctx context.Context, _tx transaction.ROTx, userID string) (*entity.User, error) {
    tx, err := mysql.ExtractROTx(_tx)
    if err != nil {
        return nil, err
    }

	var user User
	if err := tx.GetContext(ctx, &user, "SELECT * FROM users WHERE user_id = ?", userID); err != nil {
		return nil, err
	}
	return user.toEntity(), nil
}

func (r *userRepository) Update(ctx context.Context, _tx transaction.RWTx, e *entity.User) error {
    tx, err := mysql.ExtractRWTx(_tx)
    if err != nil {
    return err
    }

    if _, err := tx.ExecContext(ctx, "UPDATE users SET name = ? WHERE user_id = ?", e.Name, e.UserID); err != nil {
        return err
    }
    return nil
}
```



以上がレイヤードアーキテクチャにおけるトランザクション実装パターン「Txオブジェクトを抽象化しアプリケーション層にDIする」方法でした。
トランザクション関数とTxオブジェクトを抽象化してドメイン層に置くことで、直接的なデータベースへの依存をせずにReadOnly/ReadWriteなトランザクションのハンドリングをアプリケーション層で実現しています。

この方法のメリットは、Txオブジェクトを抽象化して各層で扱うことで、関数のシグニチャを見ただけで内部でどのようなデータベースアクセスを行うかどうかを判別できる点です（すべてのRepositoryでTxオブジェクトを受け取る前提）。
また、RepositoryのシグニチャでTxオブジェクトの抽象を受け取ることを明示的に宣言することで、予期せぬトランザクション外でのデータベースアクセスを防止できるのも利点の1つです。

デメリットを強いてあげると、トランザクション内/外で実行するRepositoryのシグニチャが異なる（Txオブジェクトを受け取るかどうか）という点です。プロジェクト内でRepository呼び出しは必ずトランザクション内で行うという合意が取れていればそこまでデメリットにならない気がしています。


## おわりに

レイヤードアーキテクチャにおけるトランザクション実装パターンを2つ紹介させていただきました。
1つ目はTxオブジェクトをContextに持たせて隠蔽する実装パターンで、Context内でTxオブジェクトの伝搬が完結することが特徴でした。
2つ目はTxオブジェクトを抽象化して各層に受け渡す実装パターンで、シグニチャを見ただけでその関数がどのようなトランザクション内での実行を期待しているかわかることが特徴でした。
プロジェクトの思想に応じて、それぞれの実装を参考にしていただければ幸いです。
