### Type class in Scala

</br>

2019-03-04

@orepuri

---

### 本日の話

</br>

型クラスとは何か

Scalaでの型クラスの実装

---

### 型クラス

</br>

アドホック多相を`後付け`で実装するための仕組み

</br>

型クラス ≠ 型

型クラス ≠ OOPのクラス

---

### アドホック多相

オブジェクトの型に応じて異なる実装を提供する

いわゆるOOPのポリモーフィズム

OOPならオーバーライドで実現

```scala
trait Animal {
  def bark: String
}

class Dog extends Animal {
  override def bark: String = "わん"
}

class Cat extends Animal {
  override def bark: String = "にゃー"
}

```
---

### 余談: パラメトリック多相

任意の型に対して同じ実装を適応する

```scala
trait List[+A] {

  def length: Int = ...

  def head: A = ...
}
```

---

### オーバーライドによるアドホック多相

</br>

クラス定義時にオーバーライドする必要がある

1つのクラスに対して1つの実装しかできない

サードパーティのライブラリなど実装を</br>
変更できない場合は適応できない

---

### 型クラスによるアドホック多相

</br>

クラス定義とは別に多相を実装できる

1つのクラスに対して複数の実装ができる

サードパーティのライブラリでも多相を実装できる

---

### 例: JSON化

```scala
case class UserId(value: Int)

case class UserName(value: String)

case class Password(value: String)

case class Email(value: String)

case class User(
  id: UserId, name: UserName, password: Password, email: Email)
```

---

### オーバーライドによる実装

```scala
trait Jsonable {
  def toJson: String
}

case class User(id: UserId, name: UserName, ...) extends Jsonable {
  // クラス定義時に多相を実装. 実装は1つだけ
  override def toJson: String = ...
}

...

val user = User(UserId(1), UserName("orepuri"), ...)
user.toJson
// { "id": 1, "name": "orepuri", "email": "xxx", ...}

```

---

### 型クラスによる実装

```scala
// 1. 型クラス. 一般的に1つの型パラメータを持つtraitで定義される
trait JsonWriter[A] {
  def write(value: A): String
}

// 2. UserクラスのJSON化の実装. 型クラスのインタンスと呼ぶ
object JsonWriterInstances {
  implicit val userJsonWriter = new JsonWriter[User] {
    def write(user: User): String = s"""{
        "id": ${user.id.value},
        "name": "${user.name.value}",
        "email": "${user.email.value}"
      }"""
  }
}
```

---

### 型クラスのインタフェース

```scala
// 3. 型クラスを使うためのインタフェース
object Json {
  def toJson[A](value: A)(implicit w: JsonWriter[A]): String = w.write(value)
}

...

import JsonWriterInstances._ // JsonWriterのインスタンスをimport

val user = User(UserId(1), UserName("orepuri"), ...)
Json.toJson(user) // implicitによりuserJsonWriterが使われる
// { "id": 1, "name": "orepuri", "email": "xxx", ...}
```

---

### 別の実装を提供

```scala
// UserクラスのJSON化の別の実装. idとnameだけを含む
object SimpleJsonWriterInstances {
  implicit val userJsonWriter = new JsonWriter[User] {
    def write(user: User): String = s"""{
        "id": ${user.id.value},
        "name": "${user.name.value}"
      }"""
  }
}
...
import SimpleJsonWriterInstances._

val user = User(UserId(1), UserName("orepuri"), ...)
Json.toJson(user)
// { "id": 1, "name": "orepuri" }
```

importするJsonWriterを返りだけで</br>
JSONフォーマット(toJsonの実装)を変更できる

注意: JsonWriterInstancesとSimpleJsonWriterInstancesは</br>
同時にimportできない
---

### シンタックス形式

もっと便利なインタフェースを提供することもできる

```scala
object JsonSyntax {
  implicit class JsonWriterOps[A](value: A) {
    def toJson(implicit w: JsonWriter[A]): String = w.write(value)
  }
}
...
import JsonWriterInstances._
import JsonSyntax._

val user = User(UserId(1), UserName("orepuri"), ...)
user.toJson
// { "id": 1, "name": "orepuri", "email": "xxx" }
```

```scala
user.toJson
=> JsonWriterOpts(user).toJson(userJsonWriter)
=> userJsonWriter.write(user)
```

---

### Scalaの型クラスの構成要素

</br>

*型クラス*</br>
 1つの型パラメータを持つtrait. (JsonWriter[A])

*型クラスのインスタンス*</br>
 型クラスの実装. (new JsonWriter[User])

*型クラスのインタフェース*</br>
 型クラスを使うためのインタフェース. (Json#toJson)

*型クラスのシンタックス*</br>
 便利なインタフェース. (JsonWriterOps)

---

### 型クラスで実装されているライブラリ
### (type-class based approch)

</br>

spray-json

scalikejdbc (TypeBinder)

rediscala (ByteStringSerializer)

Akka HTTP (Marshaller)

Cats/Scalaz

型クラスの仕組みを理解していると</br>
ライブラリの使い方も理解しやすい

---

### 型クラスの由来

</br>

プログラミング言語での最初の実装はHaskell

Haskellは言語仕様として型クラスサポート

Scalaは言語仕様としては型クラスをサポートしてないが</br>
implicitは型クラスを実現するために実装された

---

### まとめ

</br>

型クラスはアドホック多相を後付けで実装する仕組み

Scalaではtraitやimplicitを使って実装する
