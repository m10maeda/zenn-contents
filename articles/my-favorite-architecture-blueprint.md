---
title: '私のよく使うソフトウェアアーキテクチャの雛型'
emoji: '🐣'
type: 'tech'
topics:
  [
    'ソフトウェアアーキテクチャ',
    'アーキテクチャ',
    'システム設計',
    'ソフトウェア設計',
    'typescript',
  ]
published: true
---

## サンプルプロジェクト

<!-- markdownlint-disable -->

https://github.com/m10maeda/itddd

<!-- markdownlint-enable -->

## 構成

イベント駆動と CQRS を意識した、レイヤードアーキテクチャをベースとしたヘキサゴナルアーキテクチャになります。

![全体像](https://storage.googleapis.com/zenn-user-upload/4822208a6d8b-20250312.png)

### 各層について

レイヤードアーキテクチャをベースに、以下の4層に分けています。

![レイヤー構造について説明する図](https://storage.googleapis.com/zenn-user-upload/77976f29bd5d-20250312.png)

- **プレゼンテーション層:** ソフトウェアの入出力を担当
- **アプリケーション層:** ソフトウェアのユースケースを担当
- **ドメイン層:** ドメイン知識を元にしたビジネスのルールや制約、プロセスを担当
- **インフラストラクチャー層:** 技術的関心ごとの全般を担当

### ディレクトリ構成

```plain
domain/           # ドメイン層
  models/         ## ドメインモデルを格納
  services/       ## ドメインサービスを格納
application/      # アプリケーション層
  use-cases/      ## ユースケースインプットポートを格納
  interactors/    ## コマンドにあたるユースケースの実装クラスを格納
infrastructure/   # インフラストラクチャー層
  event-bus/      ## IEventPublisher の実装クラスを格納
  persistence/    ## 永続化クラス
  query-services/ ## クエリにあたるユースケースの実装クラスを格納
  messenger/      ## 外部メッセージングサービスへの発信
presentation/     # プレゼンテーション層
```

## ドメイン層

### `Event`, `IEventSubscriber`, `IEventPublisher`

![Event 周りの構成](https://storage.googleapis.com/zenn-user-upload/cef870eb6f14-20250312.png)

`Event` はドメインで起きた出来事を表すドメインオブジェクトです。

```ts
// Profile の名前を変更したことを表す Event の例
export class ProfileRenamed {
  // イベント自体の識別子
  public readonly id: ProfileEventId;

  // イベントと紐付く Profile
  public readonly profile: ProfileId;

  // イベントの発生日時
  public readonly occurredOn: Date;

  // 変更後の名前
  public readonly name: ProfileName;

  public constructor(
    id: ProfileEventId,
    profile: ProfileId,
    occurredOn: Date,
    name: ProfileName,
  ) {
    this.id = id;
    this.profile = profile;
    this.occurredOn = occurredOn;
    this.name = name;
  }
}
```

`IEventSubscriber` は `Event` を受け取った後に何らかの処理を行わせるためのインターフェースとなります。

```ts
// ProfileEvent を受け取る IEventSubscriber の例
export interface IProfileEventSubscriber {
  handle(event: ProfileEvent): Promise<void>;
}

// プロフィール名が変更されたら何らかの処理を行うクラスの例
export class DoSomethingWhenProfileRenamed implements IProfileEventSubscriber {
  public async handle(event: ProfileEvent): Promise<void> {
    if (!(event instanceof ProfileRenamed)) return;

    const { name } = event;

    // 何らかの処理
    console.log(name);
  }
}
```

`IEventPublisher` は `IEventSubscriber` へ `Event` を通知するためのインターフェースです。
`IEventPublisher` はインフラストラクチャー層で `EventBus` として実装します。

```ts
// ProfileEvent を発行する IEventPublisher の例
export interface IProfileEventPublisher {
  publish(event: ProfileEvent): Promise<void>;
}
```

### `Aggregate`

`Aggregate` は、必ず同時に変更されなければ整合性が取れなくなるデータの集まりの境界を表すドメインオブジェクトです。

`Aggregate` の状態を変更する振る舞いを呼び出した際は `Event` を返し、`Aggregate` にどのような変更が起きたのかを返すようにします。

```ts
export class Profile {
  public readonly id: ProfileId;

  public name: ProfileName;

  // 識別子によって同一かを判別
  public equals(other: Profile): boolean {
    return this.id.equals(other.id);
  }

  // 状態を変更する振る舞いは、変更されたことを示す Event を返す
  public renameTo(name: ProfileName): ProfileRenamed {
    this.name = name;

    return new ProfileRenamed(this.name);
  }

  public constructor(id: ProfileId, name: ProfileName) {
    this.id = id;
    this.name = name;
  }
}
```

### `IRepository`

`IRepository` インターフェースは `Aggregate` の取得を永続化メカニズムから隠蔽するインターフェースです。実装クラスはインフラストラクチャー層で実装します。

イベントソーシングを用いるケースでは `Aggregate` の最新状態を保存する必要がないため、保存の振る舞いは不要となります。

```ts
export interface ICircleRepository {
  findAllBy(criteria: ICircleSpecification): Promise<Iterable<Circle>>;
  getBy(id: CircleId): Promise<Circle>;
}
```

### `IFactory`

`IFactory` インターフェースは `Aggregate` の生成処理を隠蔽するインターフェースです。実装クラスはインフラストラクチャー層で実装します。

```ts
export interface IProfileFactory {
  create(name: ProfileName): Promise<Profile>;
}
```

## アプリケーション層

### `IUseCaseInputPort`

`IUseCaseInputPort` はビジネスユースケースを実現するインターフェースとなります。

個々の具体的な `IUseCaseInputPort` は以下のベースとなるインターフェースを継承し、ビジネスユースケースごとに必要な入力と出力を定義します。

```ts
export abstract class UseCaseInputData {}

export abstract class UseCaseOutputData {}

export interface IUseCaseInputPort<
  TInput extends UseCaseInputData,
  TOutput extends UseCaseOutputData,
> {
  handle(input: TInput): Promise<TOutput>;
}
```

```ts
// サークル名を変更するビジネスユースケースの入力
export class RenameCircleUseCaseInputData extends UseCaseInputData {
  // 対象のサークルID
  public readonly id: string;

  // 変更する名前
  public readonly name: string;

  public constructor(id: string, name: string) {
    super();

    this.id = id;
    this.name = name;
  }
}

// サークル名を変更するビジネスユースケースの出力
// ビジネスユースケース上、特定の出力は不要なため空のクラスとしている
export class RenameCircleUseCaseOutputData extends UseCaseOutputData {}

// サークル名を変更するビジネスユースケースの IUseCaseInputPort
export interface IRenameCircleUseCaseInputPort
  extends IUseCaseInputPort<
    RenameCircleUseCaseInputData,
    RenameCircleUseCaseOutputData
  > {}
```

各ユースケースは副作用を持つコマンド（操作）用と副作用を持たないクエリー（参照）用に分け、それぞれ `Interactor` と `QueryService` として実装します。

![コマンドとクエリーに分かれるユースケースの例](https://storage.googleapis.com/zenn-user-upload/feda9164f6c8-20250311.png)

### `Interactor`

Interactor はユースケースのうち、副作用を持つコマンド（操作）に該当するユースケースの実装クラスとなります。ドメイン層のオブジェクトを使用して一連のビジネスロジックを実現します。

![Intaractor の例](https://storage.googleapis.com/zenn-user-upload/761f22d631e5-20250311.png)

`Interactor` はドメイン層のオブジェクトを使用して、操作に関する一連のビジネスユースケースを実現します。

また `IEventPublisher` に `Event` を発行するのも `Interactor` の役割です。

```ts
// サークル名を変更するビジネスユースケースを実現する Interactor の例
export class RenameCircleInteractor implements IRenameCircleUseCaseInputPort {
  public async handle(
    input: RenameCircleUseCaseInputData,
  ): Promise<RenameCircleUseCaseOutputData> {
    // Aggregate の取得
    const circle = await this.repository.getBy(new CircleId(input.id));

    // ドメインオブジェクトを操作してビジネスロジックを実現
    const newName = new CircleName(input.name);
    const event = circle.renameTo(newName);

    // Event の発行
    await this.eventPublisher(event);

    return new RenameCircleUseCaseOutputData();
  }
}
```

## インフラストラクチャー層

### `EventBus`

`EventBus` は `IEventPublisher` を実装し、購読している `IEventSubscriber` にイベントを引渡します。

![EventBus の構成例](https://storage.googleapis.com/zenn-user-upload/f5176be55bec-20250311.png)

```ts
class EventBus implements IEventPublisher {
  private readonly subscribers: IEventSubscriber[];

  public async publish(event: Event): Promise<void> {
    await Promise.all(
      this.subscribers.map(async (subscriber) => subscriber.handle(event)),
    );
  }
}
```

### `EventStore`

`EventStore` は `IEventSubscriber` を実装し、受け取った `Event` を永続化します。

![EventStore の構成図](https://storage.googleapis.com/zenn-user-upload/9a17753759ef-20250311.png)

以下はオンメモリのスタブとなる `EventStore` の実装例です。

```ts
export class InMemoryProfileEventStore implements IProfileEventSubscriber {
  private readonly events: ProfileEvent[];

  public async handle(event: ProfileEvent): Promise<void> {
    this.events.push(event);
  }

  public constructor(events: ProfileEvent[]) {
    this.events = events;
  }
}
```

### `Repository`

`Repository` は `IRepository` の実装クラスで、蓄積された `Event` から `Aggregate` を復元して返します。

![Repository の構成例](https://storage.googleapis.com/zenn-user-upload/50640a4511e6-20250311.png)

イベントソーシングの場合、`EventStore` から `Event` を取得し、`Event` から `Aggregate` を復元します。

```ts
export interface ICircleEventLoader {
  getAllBy(id: CircleId): Promise<Iterable<CircleEvent>>;
}

export class Repository implements IRepository {
  private readonly eventLoader: ICircleEventLoader;

  public async getBy(id: CircleId): Promise<Circle> {
    const events = await this.eventLoader.getAll(id);

    const circle = this.replay(events);

    return circle;
  }

  private replay(events: CircleEvent[]): Circle | undefined {
    // CircleEvent から Circle を復元する処理
  }
}
```

### `Factory`

`Factory` はドメイン層で定義した `IFactory` の実装クラスです。

![Factory の構成例](https://storage.googleapis.com/zenn-user-upload/d1bcc3a28940-20250311.png)

### `Messenger`

`Messenger` はドメイン層で定義した `IEventSubscriber` の実装クラスです。
検知した `Event` を外部のサービスへメッセージ（MQ や Kafka など）へ通知します。

### `QueryService`

`QueryService` はユースケースのうち、副作用を持たない参照（クエリー）に該当するユースケースの実装クラスとなります。

![QueryService の構成例](https://storage.googleapis.com/zenn-user-upload/2f591b124c20-20250311.png)

```ts
// Read Model としてのサークルを取得するビジネスユースケースを実現する QueryService の実装例
export class GetCircleQueryService implements IGetCircleUseCaseInputPort {
  private readonly dao: ICircleDataAccess;

  public async handle(
    input: GetCircleUseCaseInputData,
  ): Promise<GetCircleUseCaseOutputData> {
    const { id } = input;

    // 永続化メカニズムからデータを取得
    const circle = await this.dao.getBy(id);

    return new GetCircleUseCaseOutputData(
      new CircleData(circle.id, circle.name, circle.owner, circle.members),
    );
  }
}
```

## プレゼンテーション層

### `Controller`

`Controller` はソフトウェアの入出力を担当するクラスです。受け取った入力に対応した `IUseCaseInputPort` を呼び出し、ビジネスユースケースの実行結果からレスポンスを整形して出力します。

![Controller の構成例](https://storage.googleapis.com/zenn-user-upload/cc490429daf5-20250311.png)

また外部のメッセージングサービスからのメッセージを受け取り、対応した `IUseCaseInputPort` を呼び出すのも `Controller` の責務です。

## 大まかな処理の流れ

大まかな処理の流れを簡単に表すと以下になります。

![大まかな処理の流れを説明するフロー図](https://storage.googleapis.com/zenn-user-upload/783cdd0e231e-20250311.png)

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->

コマンドにあたるビジネスユースケースを実行する場合（上図の上半分）：

<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

1. `Controller` が入力を受け付ける
2. 入力に対応した `IUseCaseInputPort`(Command) を呼び出す
3. `IUseCaseInputPort` の実装クラスの `Interactor` がドメイン層のオブジェクトを使用して一連のビジネスロジックを実行する
   1. `IRepository` を使用して Write Model の `Aggregate` を取得する
   2. `Aggregate` を操作して、変更された出来事を表す `Event` を取得する
   3. `IEventPublisher` に `Event` を発行する
4. `Controller` がレスポンスを形成して出力する

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->

クエリーにあたるビジネスユースケースを実行する場合（上図の下半分）：

<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

1. `Controller` が入力を受け付ける
2. 入力に対応した `IUseCaseInputPort`(Query) を呼び出す
3. `IUseCaseInputPort` の実装クラスの `QueryService` が永続化されたデータから定義した出力を生成する
4. `Controller` がレスポンスを形成して出力する

### データの永続化

イベントソーシングを使う場合、`Event` を蓄積していき、蓄積された `Event` を再現して `Aggregate` の最新の状態を復元します。

そのため `Aggregate` の状態を保存する必要がなくなり、ビジネスユースケース上で明示的に「保存すること」を意識せずに済みます。

## フロントエンドのアーキテクチャ

いわゆるモダンフロントエンドの場合、上記のようなアーキテクチャは適用せず、フレームワークのプラクティスに合わせます。

バックエンドがマイクロサービス化されている場合、フロントエンドはプロダクト全体のプレゼンテーションを切り出した形になることがほとんどとなります。そのため、ビジネスロジックをフロントエンドで受け持つことが少なくなることから、ビジネスロジックを隔離するためのソフトウェアアーキテクチャは不要となるケースが多くなるためです。

![前述を説明する図](https://storage.googleapis.com/zenn-user-upload/788b3d9974e9-20250312.png)
