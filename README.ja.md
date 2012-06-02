# 前置き(Prelude)

> スタイルは素晴らしいものから良いものを分離するものです。<br/>
> -- Bozhidar Batsov

このガイドの目標は、Ruby on Rails3 の開発のための
ベストプラクティスとスタイルの処方?のセットを提示することです。
それは、既存のコミュニティ主導型である [Ruby coding style guide](https://github.com/bbatsov/ruby-style-guide)
を補完するものです。

# Railsアプリケーションを開発する

## Configuration

* 初期化のためのコードは `config/initializers` に置きましょう。
  initializers におかれたコードは、アプリの起動時に実行されます。
* gemごとの初期化のためのコードは gem と同じ名前にして
  ファイルを分けましょう。例: `carrierwave.rb`, `active_admin.rb` など。
* 開発、テスト、本番環境用の設定(config/environments 下に対応するファイル)
を調整しましょう。
  * プリコンパイルのための追加assetsにはマークしましょう(もしあるのなら)

        ```Ruby
        # config/environments/production.rb
        # Precompile additional assets (application.js, application.css, and all non-JS/CSS are already added)
        config.assets.precompile += %w( rails_admin/rails_admin.css rails_admin/rails_admin.js )
        ```

* 本番環境に近い `staging` 環境の設定を作りましょう

## ルーティング(Routing)

* RESTful resource(本当に全部必要ですか?)にアクションを
  追加したくなったら、 `member` と `collection` ルートを使いましょう。

    ```Ruby
    # bad
    get 'subscriptions/:id/unsubscribe'
    resources :subscriptions

    # good
    resources :subscriptions do
      get 'unsubscribe', :on => :member
    end

    # bad
    get 'photos/search'
    resources :photos

    # good
    resources :photos do
      get 'search', :on => :collection
    end
    ```

* もし複数の `member/collection` ルートを定義する必要があるなら、
  ??のブロックを使いましょう

    ```Ruby
    resources :subscriptions do
      member do
        get 'unsubscribe'
        # more routes
      end
    end

    resources :photos do
      collection do
        get 'search'
        # more routes
      end
    end
    ```
* ネストしたルートを使うと、ActiveRecordモデルの関係を上手に表現
  することができます。
  
    ```Ruby
    class Post < ActiveRecord::Base
      has_many :comments
    end

    class Comments < ActiveRecord::Base
      belongs_to :post
    end

    # routes.rb
    resources :posts do
      resources :comments
    end
    ```

* 名前空間を使いましょう、関係のあるアクションをグルーピングできます 

    ```Ruby
    namespace :admin do
      # Directs /admin/products/* to Admin::ProductsController
      # (app/controllers/admin/products_controller.rb)
      resources :products
    end
    ```

* レガシーでワイルドなルートは決して使ってはいけません。
  このルートは全てのアクションを全てのコントローラーに対して
  GETリクエストで作成してしまうでしょう。
  
    ```Ruby
    # very bad
    match ':controller(/:action(/:id(.:format)))'
    ```

## コントローラー(Controllers)

* コントローラのぴったり感を保ち続けましょう
  - それらはビューレイヤのためにデータを検索するだけにすべきで、
    ビジネスロジックは含まれるべきでありません。
    (全てのビジネスロジックはモデルにあるのが自然でしょう)
* それぞれのコントローラーのアクションは、たった1つのメソッドを呼び出す
  べきです(理想的には)。
  初期の find または new のメソッド以外では。
* コントローラーとビューの間で2つ以上のインスタンス変数をシェアしないように
  しましょう。

## モデル(Models)

* ActiveRecordを使わないクラスは自由に????しましょう
* モデルには省略のない意味があるものを名前をつけましょう!! (ただし短く)
* ActiveRecordのふるまいをサポートしたモデルが必要になったら、
  (validation のようなやつ)
   [ActiveAttr](https://github.com/cgriego/active_attr) gem を使いましょう。

    ```Ruby
    class Message
      include ActiveAttr::Model

      attribute :name
      attribute :email
      attribute :content
      attribute :priority

      attr_accessible :name, :email, :content

      validates_presence_of :name
      validates_format_of :email, :with => /^[-a-z0-9_+\.]+\@([-a-z0-9]+\.)+[a-z0-9]{2,4}$/i
      validates_length_of :content, :maximum => 500
    end
    ```

    For a more complete example refer to the
    [RailsCast on the subject](http://railscasts.com/episodes/326-activeattr).

* 常に ["sexy" validations](http://thelucid.com/2010/01/08/sexy-validation-in-edge-rails-rails-3/) を使いましょう
* カスタムの validation を2回以上使う、または、いくつかの正規表現を使った
  validation を行うのであれば、カスタム validator を作りましょう。

    ```Ruby
    # bad
    class Person
      validates :email, format: { with: /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i }
    end

    # good
    class EmailValidator < ActiveModel::EachValidator
      def validate_each(record, attribute, value)
        record.errors[attribute] << (options[:message] || 'is not a valid email') unless value =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
      end
    end

    class Person
      validates :email, email: true
    end
    ```
* 全てのカスタム validator は共有の gem に移動すべきです
* named scope は自由に使いましょう。
* lambda とパラメータで定義された named scope はとても複雑になります。
  クラスメソッドを作ることが望ましいです。
  同じ目的で、 `ActiveRecord::Relation` オブジェクトを返す
  (良く分からず)
* `update_attribute` メソッドの挙動に注意しましょう。
  モデルの validation が動きません(`update_attributes`とは異なります)し、
  簡単にモデルの状態を破壊できます。
* ユーザに優しいURLを使いましょう。
  `id`よりも分かりやすいモデルのattributeをURLに使いましょう。
  うまくやる方法がいくつかあります。
  * モデルの `to_param` メソッドをオーバーライドしましょう。
    このメソッドは、RailsがURLからオブジェクトを構築するときに利用するものです。
    デフォルトの実装はレコードの `id` を文字列で返します。
    これは人が読みやすい attribute でオーバーライドすることができます。

        ```Ruby
        class Person
          def to_param
            "#{id} #{name}".parameterize
          end
        end
        ```
    `frendly_id` gem を使いましょう。
    `id` の代わりに、いくつかの説明的な attribute を使って、人が読みやすいURLを作ることができます。

        ```Ruby
        class Person
          extend FriendlyId
          friendly_id :name, use: :slugged
        end
        ```
        gem の詳細な情報は [gem documentation](https://github.com/norman/friendly_id) を見てください。

### ActiveResource

※ ActiveResource 知らなかったのでリンク。
  [ActiveResource の使い方（前編） : Rails 同士で通信する - WebOS Goodies](http://webos-goodies.jp/archives/how_to_use_activeresource_1.html "ActiveResource の使い方（前編） : Rails 同士で通信する - WebOS Goodies") 

* レスポンスが既存の(XMLやJSON)フォーマットと異なる、または、
  これらの形式のフォーマットに対していくつかの追加の解析が必要な場合、
  カスタムフォーマットを作り、クラスで使いましょう。
  カスタムフォーマットは、以下に上げる4つのメソッドを実装すべきです。: `extension`, `mime_type`, `encode`, そして `decode`

    ```Ruby
    module ActiveResource
      module Formats
        module Extend
          module CSVFormat
            extend self

            def extension
              'csv'
            end

            def mime_type
              'text/csv'
            end

            def encode(hash, options = nil)
              # Encode the data in the new format and return it
            end

            def decode(csv)
              # Decode the data from the new format and return it
            end
          end
        end
      end
    end

    class User < ActiveResource::Base
      self.format = ActiveResource::Formats::Extend::CSVFormat

      ...
    end
    ```

* もし、リクエストを拡張子なしで送信する必要がある場合は、
  `ActiveResource::Base` の `element_path` と `collection_path` メソッドを
  オーバーライドして、拡張子のところを削除しましょう。

    ```Ruby
    class User < ActiveResource::Base
      ...

      def self.collection_path(prefix_options = {}, query_options = nil)
        prefix_options, query_options = split_options(prefix_options) if query_options.nil?
        "#{prefix(prefix_options)}#{collection_name}#{query_string(query_options)}"
      end

      def self.element_path(id, prefix_options = {}, query_options = nil)
        prefix_options, query_options = split_options(prefix_options) if query_options.nil?
        "#{prefix(prefix_options)}#{collection_name}/#{URI.parser.escape id.to_s}#{query_string(query_options)}"
      end
    end
    ```

    他にURLを変更する必要がある場合でも、これらのメソッドをオーバーライド
    することができます。

### マイグレーション(Migrations)

* `schema.rb` をバージョン管理しましょう。
* 空っぽのデータベースを初期化する場合は、 `rake db:migrate` の代わりに
  `rake db:schema:load` を使いましょう。
* テストのデータベースのスキーマを更新する場合は、 `rake db:test:prepare` を
  使いましょう。
* テーブル自体にデフォルトの値を設定しないようにしましょう。
  代わりにモデルのレイヤーを使いましょう。

    ```Ruby
    def amount
      self[:amount] or 0
    end
    ```

  `self[:attr_name]` を利用することは慣用的です。
  代わりにやや冗長な（そしておそらくより読みやすい）read_attributeを
  使うことを検討したほうがよいでしょう。

    ```Ruby
    def amount
      read_attribute(:amount) or 0
    end
    ```

* 建設的なマイグレーション(テーブルや列を追加する)を書いている場合は、
  新しい Rails 3.1 の方法を使いましょう。
  - `up` や `down` メソッドの代わりに `change` メソッドを使いましょう。

    ```Ruby
    # the old way
    class AddNameToPerson < ActiveRecord::Migration
      def up
        add_column :persons, :name, :string
      end

      def down
        remove_column :person, :name
      end
    end

    # the new prefered way
    class AddNameToPerson < ActiveRecord::Migration
      def change
        add_column :persons, :name, :string
      end
    end
    ```

## Views

* ビューから直接モデルのレイヤーを呼び出してはいけません。
* ビューの中で複雑なフォーマットを作ってはいけません。
  フォーマットのメソッドはヘルパーか、モデルに外だししましょう。
* partial テンプレートや、レイアウトを使ってコードの重複を軽減しましょう。
* カスタムのValidatorのために [client side validation](https://github.com/bcardarella/client_side_validations) を追加しましょう。
  手順は次のとおりです。:
  * `ClientSideValidations::Middleware::Base` を継承したカスタム validator を定義します。

        ```Ruby
        module ClientSideValidations::Middleware
          class Email < Base
            def response
              if request.params[:email] =~ /^([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})$/i
                self.status = 200
              else
                self.status = 404
              end
              super
            end
          end
        end
        ```

  * 新しく `public/javascripts/rails.valiations.custom.js.coffee`
    を作成して、 `application.js.coffee` に参照を追加します。

        ```Ruby
        # app/assets/javascripts/application.js.coffee
        #= require rails.validations.custom
        ```
  * Client-side の validator を追加します。

        ```Ruby
        #public/javascripts/rails.validations.custom.js.coffee
        clientSideValidations.validators.remote['email'] = (element, options) ->
          if $.ajax({
            url: '/validators/email.json',
            data: { email: element.val() },
            async: false
          }).status == 404
            return options.message || 'invalid e-mail format'
        ```

## 国際化(Internationalization)



