<!--

## Versions {-}

This book is written for Scala @SCALA_VERSION@ and Cats @CATS_VERSION@.
Here is a minimal `build.sbt` containing
the relevant dependencies and settings[^sbt-version]:

```scala
scalaVersion := "@SCALA_VERSION@"

libraryDependencies +=
  "org.typelevel" %% "cats-core" % "@CATS_VERSION@"

scalacOptions ++= Seq(
  "-Xfatal-warnings"
)
```

[^sbt-version]: We assume you are using SBT 1.0.0 or newer.

### Template Projects {-}

For convenience, we have created
a Giter8 template to get you started.
To clone the template type the following:

```bash
$ sbt new scalawithcats/cats-seed.g8
```

This will generate a sandbox project
with Cats as a dependency.
See the generated `README.md` for
instructions on how to run the sample code
and/or start an interactive Scala console.

The `cats-seed` template is very minimal.
If you'd prefer a more batteries-included starting point,
check out Typelevel's `sbt-catalysts` template:

```bash
$ sbt new typelevel/sbt-catalysts.g8
```

This will generate a project with a suite
of library dependencies and compiler plugins,
together with templates for unit tests
and documentation.
See the project pages for [catalysts][link-catalysts]
and [sbt-catalysts][link-sbt-catalysts]
for more information.


```scala mdoc:reset:silent
```
--->

## バージョンについて {-}

本書は Scala @SCALA_VERSION@ および Cats @CATS_VERSION@ を対象として書かれている。
以下に、必要な依存関係と設定を含んだ最小限の `build.sbt` を示す[^sbt-version]。

```scala
scalaVersion := "@SCALA_VERSION@"

libraryDependencies +=
  "org.typelevel" %% "cats-core" % "@CATS_VERSION@"

scalacOptions ++= Seq(
  "-Xfatal-warnings"
)
```

[^sbt-version]: SBT 1.0.0 以降を使用していることを前提とする。

### テンプレートプロジェクト {-}

準備の手間を省けるよう Giter8 テンプレートを用意している。以下のコマンドでテンプレートをクローンできる。

```bash
$ sbt new scalawithcats/cats-seed.g8
```

このコマンドにより、依存ライブラリとして Cats を含んだサンドボックスプロジェクトが生成される。生成された `README.md` を参照すれば、サンプルコードの実行方法や、インタラクティブな Scala コンソールの起動手順を確認できる。

この `cats-seed` テンプレートは最小限の構成になっている。もっと多くのライブラリやツールを含んだ構成から始めたい場合は、Typelevel の `sbt-catalysts` テンプレートを利用するとよいだろう。

```bash
$ sbt new typelevel/sbt-catalysts.g8
```

このコマンドにより、単体テストやドキュメントのテンプレートに加えて、複数の依存ライブラリやコンパイラプラグインを備えたプロジェクトが生成される。詳細については [catalysts][link-catalysts] および [sbt-catalysts][link-sbt-catalysts] のプロジェクトページを参照してほしい。
