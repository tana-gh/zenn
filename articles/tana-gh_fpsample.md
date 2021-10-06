---
title: "再利用性を高めるコーディング方法メモ【C#, Haskell, Scalaを比較】"
emoji: "💘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "関数型", "csharp", "haskell", "scala" ]
published: true
---
# はじめに

本記事では、**再利用性**に焦点を絞ってコーディング方法を考察します。
以下の3つの言語で簡単なプログラムを作成し、比較します。

- C#
- Haskell
- Scala

# 本記事が対象とするアプリの仕様

1. HTTPサーバーから以下の形式のJSONを取得する。

```json
{
  "name": "John Smith",
  "age": 20
}
```

2. 取得したデータを標準出力に書き出す。

```text
name=John Smith
age=20
```

# 本記事における再利用性

**テストにおいてアプリのコードを再利用します**。
ここで、テストは次のように行ないます。

- 前節 1. において、HTTPサーバーからJSONを取得するコードはテストしない。
- 前節 2. において、標準出力に正常に書き出せたかどうかはテストしない。
- テストするのは、1. の処理が呼ばれ、次に 2. の処理が呼ばれたことの確認のみとする。

つまり、**実装の詳細部分は無視し、アプリケーションの処理フローが正しいことのみをテストする**ということです。

# サンプルプログラム

以下のGitHubリポジトリを参照してください。

[https://github.com/tana-gh/fpsample](https://github.com/tana-gh/fpsample)

# C#

C#はオブジェクト指向言語色が濃い言語です。
再利用性という観点では、インターフェイスとDIを利用してゆくことになります。

- アプリケーションドメインでは、機能詳細をインターフェイスで抽象化する。
- 機能詳細インターフェイスの実装をアプリ版とテスト版の2種類用意する。
- アプリまたはテストの実行時、機能詳細の実装クラスをDIで注入する。

## アプリケーションドメイン

`fpsample-csharp/fpsample-csharp.Lib/`以下のコードです。

### データの表現

```csharp:Lib/Data.cs
namespace fpsample_csharp.Lib
{
    public record Data(string Name, int Age);  // NameとAgeがある
}
```

アプリに入出力されるデータの表現です。

### 機能詳細インターフェイス

```csharp:Lib/IDataReader.cs
using System.Threading.Tasks;

namespace fpsample_csharp.Lib
{
    public interface IDataReader
    {
        Task<Data> Read();  // データを取得するメソッド
    }
}
```

```csharp:Lib/IDataWriter.cs
using System.Threading.Tasks;

namespace fpsample_csharp.Lib
{
    public interface IDataWriter
    {
        Task Write(Data data); // データを出力するメソッド
    }
}
```

機能詳細は`Read`と`Write`のみです。
インターフェイスで表現します。

### アプリケーションの処理フロー

```csharp:Lib/Application.cs
using System.Threading.Tasks;

namespace fpsample_csharp.Lib
{
    public class Application
    {
        private IDataReader Reader { get; }
        private IDataWriter Writer { get; }

        public Application(IDataReader reader, IDataWriter writer)  // DI
        {
            Reader = reader;
            Writer = writer;
        }

        public async Task Run()  // アプリケーションの本処理
        {
            var data = await Reader.Read();
            await Writer.Write(data);
        }
    }
}
```

アプリケーションの処理としては、`Read`を呼んでから`Write`を呼ぶという流れが重要であり、これがテスト対象となります。

## 機能詳細

`fpsample-csharp/fpsample-csharp.App/`以下のコードです。

### インターフェイスの実装

```csharp:App/DataReader.cs
using System;
using System.Net.Http;
using System.Text.Json;
using System.Threading.Tasks;
using fpsample_csharp.Lib;

namespace fpsample_csharp.App
{
    internal class DataReader : IDataReader
    {
        private static HttpClient HttpClient { get; } = new();

        private string Url { get; }

        public DataReader(string url)
        {
            Url = url;
        }

        public async Task<Data> Read()
        {
            try
            {
                using var res = await HttpClient.GetAsync(Url);  // HTTPサーバーから取得
                var stream = await res.Content.ReadAsStreamAsync();
                return await JsonSerializer.DeserializeAsync<Data>  // JSONをデコード
                (
                    stream,
                    new JsonSerializerOptions() { PropertyNameCaseInsensitive = true }
                );
            }
            catch
            {
                Console.Error.WriteLine("Fail to read data.");
                throw;
            }
        }
    }
}
```

```csharp:App/DataWriter.cs
using System;
using System.Threading.Tasks;
using fpsample_csharp.Lib;

namespace fpsample_csharp.App
{
    internal class DataWriter : IDataWriter
    {
        public Task Write(Data data)
        {
            Console.WriteLine($"name={data.Name}");  // 標準出力
            Console.WriteLine($"age={data.Age}");
            return Task.CompletedTask;
        }
    }
}
```

### アプリの実行

```csharp:App/Program.cs
using System.Threading.Tasks;
using CommandLine;
using fpsample_csharp.Lib;

namespace fpsample_csharp.App
{
    internal class Options  // コマンドラインオプションの情報
    {
        [Option("url", Required = true, HelpText = "Set url for reading data.")]
        public string Url { get; set; }
    }

    internal class Program
    {
        private static Task Main(string[] args)
        {
            return Parser.Default.ParseArguments<Options>(args).WithParsedAsync<Options>(o =>
            {
                var reader = new DataReader(o.Url);
                var writer = new DataWriter();
                var app    = new Application(reader, writer);  // DI
                return app.Run();  // 本処理実行
            });
        }
    }
}
```

## テスト

`fpsample-csharp/fpsample-csharp.Test/`以下のコードです。

### テストの状態

```csharp:Test/TestState.cs
using System.Collections.Generic;

namespace fpsample_csharp.Test
{
    internal enum TestEvent
    {
        ReadCalled,  // Readが呼ばれた
        WriteCalled  // Writeが呼ばれた
    }

    internal class TestState
    {
        public List<TestEvent> Events { get; } = new();  // 上記イベントをListに保持
    }
}
```

### インターフェイスの実装

```csharp:Test/TestReader.cs
using System.Threading.Tasks;
using fpsample_csharp.Lib;

namespace fpsample_csharp.Test
{
    internal class TestReader : IDataReader
    {
        private TestState State { get; set; }

        public TestReader(TestState state)
        {
            State = state;
        }

        public Task<Data> Read()
        {
            State.Events.Add(TestEvent.ReadCalled);  // イベントを記録
            return Task.FromResult(new Data("foo", 1));
        }
    }
}
```

```csharp:Test/TestWriter.cs
using System.Threading.Tasks;
using fpsample_csharp.Lib;

namespace fpsample_csharp.Test
{
    internal class TestWriter : IDataWriter
    {
        private TestState State { get; set; }

        public TestWriter(TestState state)
        {
            State = state;
        }

        public Task Write(Data data)
        {
            State.Events.Add(TestEvent.WriteCalled);  // イベントを記録
            return Task.CompletedTask;
        }
    }
}
```

### テストケースの実装

```csharp:Test/UnitTest.cs
using System.Threading.Tasks;
using Xunit;
using fpsample_csharp.Lib;

namespace fpsample_csharp.Test
{
    public class UnitTest
    {
        [Fact]
        public async Task Test()
        {
            var state  = new TestState();
            var reader = new TestReader(state);
            var writer = new TestWriter(state);
            var app    = new Application(reader, writer);

            await app.Run();  // 本処理実行

            Assert.Equal  // イベント呼び出しのアサーション
            (
                new TestEvent[]
                {
                    TestEvent.ReadCalled,
                    TestEvent.WriteCalled
                },
                state.Events.ToArray()
            );
        }
    }
}
```

## 考察

C#はオブジェクト指向言語ですが、状態を持たないオブジェクトを多用することにより、関数型的な書き方を実現できます。
このように、変更可能な状態を持たず関数だけが定義されたオブジェクトを**Functional Object**と呼ぶことがあるようです。
Functional Objectは、インターフェイスとDIを利用することで、簡単に機能の切り替えが可能です。
これにより、疎結合と再利用性を確保しています。

# Haskell

Haskellは純粋関数型言語です。
純粋関数と強力な型で表現されるコードの再利用性を高めるために、モナド型クラスを利用することができます。

- アプリケーションドメインでは、型クラスを定義し、その型の上で処理を記述する。
- 機能詳細型クラスの実装をアプリ版とテスト版の2種類用意する。
- アプリまたはテストの実行時、型クラスを実装した型をコンテキストとして置く。

## アプリケーションドメイン

`fpsample-haskell/src/FpsampleHaskell/`以下のコードです。

### データの表現

```haskell:Data.hs
{-# LANGUAGE DeriveGeneric #-}

module FpsampleHaskell.Data
    ( Data(..)
    ) where

import GHC.Generics
    ( Generic
    )

data Data = Data
    { name :: String
    , age  :: Int
    } deriving Generic  -- AesonのためにGenericとする
```

アプリに入出力されるデータの表現です。

### 機能詳細型クラス

```haskell:Monad.hs
module FpsampleHaskell.Monad
    ( MonadDataReader(readData)
    , MonadDataWriter(writeData)
    , MonadApp
    ) where

import FpsampleHaskell.Data
    ( Data
    )

class Monad m => MonadDataReader m where
    readData :: m Data  -- データを取得する関数

class Monad m => MonadDataWriter m where
    writeData :: Data -> m ()  -- データを出力する関数

class (MonadDataReader m, MonadDataWriter m) => MonadApp m  -- 上記をまとめたもの
```

型クラスはモナドとして定義します。

### アプリケーションの処理フロー

```haskell:App.hs
module FpsampleHaskell.App
    ( app
    ) where

import FpsampleHaskell.Monad
    ( MonadApp
    , MonadDataReader(..)
    , MonadDataWriter(..)
    )

app :: (MonadApp m) => m ()
app = readData >>= writeData  -- 処理順を定義
```

`MonadApp`はモナドなので、手続き的に順序立てて処理することが可能です。

## 機能詳細

`fpsample-haskell/app/`以下のコードです。

### アプリの型定義と実行

```haskell:Main.hs
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE GeneralisedNewtypeDeriving #-}

module Main where

import Control.Monad.IO.Class
    ( MonadIO(..)
    )
import Control.Monad.Reader
    ( MonadReader
    , MonadTrans
    , ReaderT(..)
    , asks
    , runReaderT
    )
import Data.Aeson
    ( FromJSON
    )
import Network.HTTP.Simple
    ( Response
    , parseRequest
    , getResponseBody
    , httpJSON
    )
import Options.Applicative
    ( (<**>)
    , execParser
    , fullDesc
    , header
    , help
    , helper 
    , info
    , long
    , progDesc
    , strOption
    )
import Options.Applicative.Types
    ( Parser
    )
import FpsampleHaskell.App
    ( app
    )
import FpsampleHaskell.Data
    ( Data(..)
    )
import FpsampleHaskell.Monad
    ( MonadApp
    , MonadDataReader(..)
    , MonadDataWriter(..)
    )

newtype App r m a = App  -- MonadAppの実体
    { runApp :: ReaderT r m a
    } deriving
    ( Functor
    , Applicative
    , Monad
    , MonadTrans
    , MonadReader r
    , MonadIO
    )

newtype Config = Config  -- アプリの設定
    { url :: String
    }

instance FromJSON Data  -- DataにFromJSONを実装

instance MonadIO m => MonadDataReader (App Config m) where
    readData = do
        url' <- asks url
        req  <- liftIO $ parseRequest url'
        res  <- liftIO $ httpJSON req  -- HTTPサーバーから取得し、同時にJSONをデコード
        return $ getResponseBody res
        
instance MonadIO m => MonadDataWriter (App Config m) where
    writeData data' =
        liftIO $ do
            putStrLn $ "name=" ++ name data'  -- 標準出力
            putStrLn $ "age=" ++ show (age data')

instance MonadIO m => MonadApp (App Config m)

newtype Options = Options  -- コマンドラインオプションの情報
    { urlOption :: String
    }

parser :: Parser Options
parser = Options <$> strOption (long "url" <> help "Set url for reading data.")

main :: IO ()
main = do
    options <- execParser $ info (parser <**> helper) fullDesc
    let config = Config $ urlOption options
    (`runReaderT` config) $ runApp app  -- 本処理実行
```

## テスト

`fpsample-haskell/test/`以下のコードです。

### テストの型定義とテストケースの実装

```haskell:Test.hs
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE GeneralisedNewtypeDeriving #-}

module Main where

import Control.Monad.State
    ( MonadIO
    , MonadState
    , MonadTrans
    , StateT(..)
    , execStateT
    , modify
    )
import Test.Hspec
    ( Spec
    , hspec
    , describe
    , it
    , shouldBe
    )
import FpsampleHaskell.App
    ( app
    )
import FpsampleHaskell.Data
    ( Data(..)
    )
import FpsampleHaskell.Monad
    ( MonadApp
    , MonadDataReader(..)
    , MonadDataWriter(..)
    )

newtype TestApp s m a = TestApp  -- MonadAppの実体（テスト用）
    { runApp :: StateT s m a
    } deriving
    ( Functor
    , Applicative
    , Monad
    , MonadTrans
    , MonadState s
    , MonadIO
    )

data TestEvent
    = ReadDataCalled   -- readDataが呼ばれた
    | WriteDataCalled  -- writeDataが呼ばれた
    deriving (Eq, Show)

newtype TestState = TestState [TestEvent]  -- 上記イベントを保持するための型

instance Monad m => MonadDataReader (TestApp TestState m) where
    readData = do
        modify $ \(TestState events) -> TestState (ReadDataCalled : events)  -- イベントを記録
        return Data { name = "foo", age = 1 }
        
instance Monad m => MonadDataWriter (TestApp TestState m) where
    writeData _ = modify $ \(TestState events) -> TestState (WriteDataCalled : events)  -- イベントを記録

instance Monad m => MonadApp (TestApp TestState m)

spec :: Spec
spec =
    describe "app" $
        it "readData and writeData are called correctly" $ do
            (TestState events) <- (`execStateT` TestState []) $ runApp app  -- 本処理実行
            events `shouldBe` [ WriteDataCalled, ReadDataCalled ]  -- イベント呼び出しのアサーション

main :: IO ()
main = hspec spec
```

## 考察

Haskellは型が非常に強力で、副作用を型の力で分離することができます。
それを実現するためにモナドを多用することになるため、再利用性の確保にはモナド型クラスを使用するのが1つの方策となります。

# Scala

Scalaはオブジェクト指向と関数型を絶妙なバランスで融合させているような言語です。
型機能が強力であるため、Haskellのようにモナドを使用して記述することもできますが、ここではもう少しソフトな方法を試します。
Scalaは基本的には手続き型ですので、副作用に関しては型クラスに盛り込まないこととします。
そのため、再利用性の確保にモナドは利用しません。

- アプリケーションドメインでは、型クラスを定義し、その型の上で処理を記述する。
- 機能詳細型クラスの実装をアプリ版とテスト版の2種類用意する。
- アプリまたはテストの実行時、`given`オブジェクトを用意する。

## アプリケーションドメイン

`fpsample-scala/lib/src/main/scala/`以下のコードです。

### データの表現

```scala:Data.scala
package fpsample_scala.lib

case class Data(name: String, age: Int)  // nameとageがある
```

アプリに入出力されるデータの表現です。

### 機能詳細型クラス

```scala:Traits.scala
package fpsample_scala.lib

import scala.concurrent.{
  ExecutionContext,
  Future
}

trait DataReader[App]:
  extension (app: App)
    def read(using ExecutionContext): Future[Data]  // データを取得する関数

trait DataWriter[App]:
  extension (app: App)
    def write(data: Data)(using ExecutionContext): Future[Unit]  // データを出力する関数
```

ここでは型クラスはモナドではなくただの制約です。

### アプリケーションの処理フロー

```scala:App.scala
package fpsample_scala.lib

import scala.concurrent.{
  ExecutionContext,
  Future
}

def runApp[App](app: App)(using DataReader[App], DataWriter[App], ExecutionContext): Future[Unit] =  // アプリの本処理
  for
    data <- app.read
    ()   <- app.write(data)
  yield ()
```

`Future`があるのでモナド的に処理していますが、`App`型自体はモナドではありません。

## 機能詳細

`fpsample-scala/app/src/main/scala/`以下のコードです。

### アプリの型定義

```scala:Impl.scala
package fpsample_scala.app

import scala.compat.java8.{
  FutureConverters
}
import scala.concurrent.{
  ExecutionContext,
  Future
}
import java.net.{
  URI
}
import java.net.http.{
  HttpClient,
  HttpResponse,
  HttpRequest
}
import io.circe.generic.auto.*
import io.circe.parser.{
  decode
}
import fpsample_scala.lib.{
  Data,
  DataReader,
  DataWriter
}

case class App(url: String)  // アプリ型

given DataReader[App] with
  extension (app: App)
    def read(using ExecutionContext): Future[Data] =
      val client = HttpClient.newHttpClient()
      val req    = HttpRequest
        .newBuilder
        .uri(URI.create(app.url))
        .build
      
      for
        res <- FutureConverters.toScala(
          client
            .sendAsync(req, HttpResponse.BodyHandlers.ofString())  // HTTPサーバーから取得
        )
      yield
        decode[Data](res.body) match  // JSONをデコード
          case Left(e) =>
            Console.err.println(e)
            throw IllegalStateException()
          case Right(data) => data

given DataWriter[App] with
  extension (app: App)
    def write(data: Data)(using ExecutionContext): Future[Unit] =
      println(s"name=${data.name}")  // 標準出力
      println(s"age=${data.age}")
      Future.successful(())
```

Scalaでは、型クラスの実装を`given`オブジェクトとして定義しておきます。

### アプリの実行

```scala:Main.scala
package fpsample_scala.app

import scala.concurrent.{
  Await
}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global
import scala.language.{
  postfixOps
}
import scopt.{
  OParser
}
import fpsample_scala.lib.{
  runApp
}

case class Options(url: String = "")  // コマンドラインオプションの情報

@main
def main(args: String*): Unit = 
  val builder = OParser.builder[Options]
  val parser  = OParser.sequence(
    builder.opt[String]("url")
      .required()
      .action((x, opt) => opt.copy(url = x))
      .text("Set url for reading data.")
  )

  OParser.parse(parser, args, Options()) match
    case Some(options) =>
      Await.ready(runApp(App(options.url)), Duration.Inf)  // 本処理実行
    case _ => ()
```

## テスト

`fpsample-scala/app/src/test/scala/`以下のコードです。

### テストの型定義とテストケースの実装

```scala:Test.scala
import scala.collection.mutable.ListBuffer
import scala.concurrent.{
  Await,
  ExecutionContext,
  Future
}
import scala.concurrent.duration.*
import scala.concurrent.ExecutionContext.Implicits.global
import scala.language.{
  postfixOps
}
import org.scalatest.funsuite.{
  AnyFunSuite
}
import fpsample_scala.lib.{
  Data,
  DataReader,
  DataWriter,
  runApp
}


enum TestEvent:
  case ReadCalled   // readが呼ばれた
  case WriteCalled  // writeが呼ばれた

import TestEvent.*

case class TestState(
  events: ListBuffer[TestEvent] = ListBuffer[TestEvent]()  // 上記イベントをListに保持
)

case class TestApp(state: TestState)  // アプリ型（テスト用）

given DataReader[TestApp] with
  extension (app: TestApp)
    def read(using ExecutionContext): Future[Data] =
      app.state.events += ReadCalled  // イベントを記録
      Future.successful(Data("foo", 1))

given DataWriter[TestApp] with
  extension (app: TestApp)
    def write(data: Data)(using ExecutionContext): Future[Unit] =
      app.state.events += WriteCalled  // イベントを記録
      Future.successful(())

class SetSuite extends AnyFunSuite:
  test("readData and writeData are called correctly") {
    val state = TestState()
    Await.ready(runApp(TestApp(state)), Duration.Inf)  // 本処理実行
    assertResult(List(ReadCalled, WriteCalled))(state.events.toList)  // イベント呼び出しのアサーション
  }
```

## 考察

Scalaには cats や zio などの強力な型ライブラリがありますが、それらを使わずにコーディングするとこうなりました。
型情報で副作用を分離してはいないのですが、再利用性を高める1つの動機として副作用の分離は重要であるため、型の強さに関わらずコーダーは副作用は分離するはずです。
もちろん、型ライブラリを使用すれば副作用をコンパイル時点で検証できますが、本記事ではそこまでやりませんでした。
ここで示したコードではあまりメリットが感じませんが、Scalaは手続き型の記述ができるため、このようなやり方でも恩恵があると思います。

# まとめ

3つの言語について再利用性を意識したコード例を示しました。
これ以上無いくらい単純な例でしたが、このやり方は実践でも役に立つと思います。
本記事で示したやり方以外にも方法はありますが、シンプルさを重視してこのような形になりました。
非常に長くなりましたが、ご覧いただきありがとうございました。
