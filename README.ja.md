# 前置き(Prelude)

> スタイルは素晴らしいものから良いものを分離するものです。<br/>
> -- Bozhidar Batsov

このガイドの目標は、Ruby on Rails3 の開発のための
ベストプラクティスとスタイルの処方?のセットを提示することです。
それは、既存のコミュニティ主導型である [Ruby coding style guide](https://github.com/bbatsov/ruby-style-guide)
を補完するものです。

ガイドのセクション [Testing Rails applications](#testing) は、
[Developing Rails applications](#developing) の後にありますが、
私は [Behaviour-Driven Development](http://en.wikipedia.org/wiki/Behavior_Driven_Development) を本当に信じています。
(BDD) はソフトウェア開発の最善の道です。このことを念頭においてください。

Railsは独断的なフレームワーク(って何??)であり、これは独断的なガイドです。
私は、
[RSpec](https://www.relishapp.com/rspec) は Test::Unit よりも素晴らしい
[Sass](http://sass-lang.com/) は CSS よりも素晴らしい、そして、
[Haml](http://haml-lang.com/) ([Slim](http://slim-lang.com/)) は Erb よりも素晴らしい
と思っています。
なので、 Test::Unit, CSS, Erb のアドバイスはここにはありません。

いくつかのアドバイスは Rails 3.1+ に適用されます。

あなたは [Transmuter](https://github.com/TechnoGate/transmuter) を使って
PDF または HTML のコピーを作ることができます。

# Table of Contents

* [Developing Rails applications](#developing-rails-applications)
    * [Configuration](#configuration)
    * [Routing](#routing)
    * [Controllers](#controllers)
    * [Models](#models)
    * [Migrations](#migrations)
    * [Views](#views)
    * [Assets](#assets)
    * [Mailers](#mailers)
    * [Bundler](#bundler)
    * [Priceless Gems](#priceless-gems)
    * [Flawed Gems](#flawed-gems)
    * [Managing processes](#managing-processes)
* [Testing Rails applications](#testing-rails-applications)
    * [Cucumber](#cucumber)
    * [RSpec](#rspec)

# Developing Rails applications

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

## Routing

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

## Controllers

* コントローラのぴったり感を保ち続けましょう
  - それらはビューレイヤのためにデータを検索するだけにすべきで、
    ビジネスロジックは含まれるべきでありません。
    (全てのビジネスロジックはモデルにあるのが自然でしょう)
* それぞれのコントローラーのアクションは、たった1つのメソッドを呼び出す
  べきです(理想的には)。
  初期の find または new のメソッド以外では。
* コントローラーとビューの間で2つ以上のインスタンス変数をシェアしないように
  しましょう。

## Models

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

### Migrations

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

## Internationalization

* 文字列やその他ロケール固有の設定はビュー、モデル、コントローラーで
  利用すべきではありません。
  これらのテキストは `config/locale` のロケールファイルに移動すべきです。
* ActiveRecord のラベルを翻訳する必要がある場合は、
  `activerecord` スコープを使いましょう。

    ```
    en:
      activerecord:
        models:
          user: Member
        attributes:
          user:
            name: "Full name"
    ```

    `User.model_name.human` は "Member" そして `User.human_attribute_name("name")`は、
    "Ful name" を返すでしょう。

* ビューで使われているテキストを ActiveRecord の attributes の翻訳から分割しましょう。
  ロケールファイルを置きましょう。(以下分からず)
  * ロケールファイルのまとまりが追加のディレクトリで行われている場合(ってなんだ?)、これらのディレクトリのファイルをロードするためには、 `application.rb` ファイルに記述する必要があります。

        ```Ruby
        # config/application.rb
        config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}')]
        ```
* `locales` ディレクトリのルートに、日付や通貨の書式などの共有ローカライズオプションを置きましょう。
* `I18n.translate` の代わりに短縮形の `I18n.t` を使いましょう。
  `I18n.localize` の代わりに `I18n.l` を使いましょう。
* ビューで利用されるテキストのために "lazy" ルックアップを使いましょう。
  次のような構造になっています。

    ```
    en:
      users:
        show:
          title: "User details page"
    ```

    `users.show.title` の値は、テンプレート `app/views/users/show.html.haml` で
    ルックアップされます。

    ```Ruby
    = t '.title'
    ```

* コントローラーやモデルでは `:scope` オプションで指定する代わりに、
  ドットで区切られたキーを使いましょう。

    ```Ruby
    # こっちを使う
    I18n.t 'activerecord.errors.messages.record_invalid'

    # これの代わりに
    I18n.t :record_invalid, :scope => [:activerecord, :errors, :messages]
    ```

* Railsのi18nに関する詳細な情報は  [Rails
Guides](http://guides.rubyonrails.org/i18n.html) にあります。

## Assets

アプリケーションをよりまとまったものにするために、
[assets pipeline](http://guides.rubyonrails.org/asset_pipeline.html) を使いましょう。

* カスタムの stylesheets, javascript, または images のために `app/assets` を使いましょう。
* [jQuery](http://jquery.com/) や [bootstrap](http://twitter.github.com/bootstrap/) のようなサードパーティのコードは、 `vendor/assets` に置くべきです。
* 可能であれば、 assets の gemified バージョンを使いましょう。
  (例. [jquery-rails](https://github.com/rails/jquery-rails))
  # gem を作れって事????

## Mailers

* `SomethingMailer` のようにメーラーに名前をつけましょう。
  メーラーの接尾語がなければ、どのビューがメーラーに依存しているのか
  すぐに分かりません。
* HTMLとプレインテキストのテンプレートを提供しましょう。
* 開発環境ではメールの配送が失敗した場合に、エラーがあがるようにしましょう。
  デフォルトではエラーがあがりません。

    ```Ruby
    # config/environments/development.rb

    config.action_mailer.raise_delivery_errors = true
    ```

* 開発環境のSMTPサーバとして、 `smtp.gmail.com` を使いましょう。
  (ローカルのSMTPサーバを持っていないなら)
 
    ```Ruby
    # config/environments/development.rb

    config.action_mailer.smtp_settings = {
      address: 'smtp.gmail.com',
      # more settings
    }
    ```

* ホスト名のデフォルト設定を提供しましょう。

    ```Ruby
    # config/environments/development.rb
    config.action_mailer.default_url_options = {host: "#{local_ip}:3000"}


    # config/environments/production.rb
    config.action_mailer.default_url_options = {host: 'your_site.com'}

    # in your mailer class
    default_url_options[:host] = 'your_site.com'
    ```

* メールにサイトのリンクを使う必要がある場合は、常に `_url` を使い、
  `_path` メソッドは使わないようにしましょう。
  `_url` メソッドはホスト名を含んでいますが、 `_path` メソッドは含んでいません。

    ```Ruby
    # wrong
    You can always find more info about this course
    = link_to 'here', url_for(course_path(@course))

    # right
    You can always find more info about this course
    = link_to 'here', url_for(course_url(@course))
    ```

* from と to アドレスの書式を設定しましょう。
  以下のようなフォーマットを使いましょう。

    ```Ruby
    # in your mailer class
    default from: 'Your Name <info@your_site.com>'
    ```

* テスト環境で使うメール配信用のメソッドをはっきりさせましょう。
  `test` に設定しましょう。

    ```Ruby
    # config/environments/test.rb

    config.action_mailer.delivery_method = :test
    ```

  開発と本番環境で使う配信用のメソッドは `smtp` にすべきです。

    ```Ruby
    # config/environments/development.rb, config/environments/production.rb

    config.action_mailer.delivery_method = :smtp
    ```

* いくつかのメールクライアントが外部のスタイルに問題があるとして、
  HTMLメールを送信するときにすべてのスタイルは、インラインでなければいけません。
  しかし、これは、メンテナンスを難しくし、コードの重複につながります。
  スタイルを変換し、対応するHTMLタグを挿入する2つの似たようなgemがあります。
  [premailer-rails3](https://github.com/fphilipe/premailer-rails3) と、
  [roadie](https://github.com/Mange/roadie) です。

* ページを生成している間にメールを送信するのは、避けるべきです。
  これはページのロードを遅らせ、複数のメールを送った場合は、リクエストが
  タイムアウトするでしょう。
  メールをバックグラウンドで送信することで、回避します。
   [delayed_job](https://github.com/tobi/delayed_job) gem の助けを借りましょう。

## Bundler

* 開発、テストだけで使う gem は Gemfile の適切なグループに入れましょう。
* 自分のプロジェクトで確立された gem を使いましょう(どういう意味?)
  あまり知られていない gem を含め考慮しているのであれば、まず、
  それらのソースコードを慎重にレビューすべきです。
* OS固有の gem (良く分からず???)
  全ての OS X 固有の gem は `darwin` グループに入れましょう。
  そして、Linux 固有の gem は `linux` グループに入れましょう。

    ```Ruby
    # Gemfile
    group :darwin do
      gem 'rb-fsevent'
      gem 'growl'
    end

    group :linux do
      gem 'rb-inotify'
    end
    ```

    正しい環境で適切な gem を必要とするのであれば、
    `config/application.rb` に以下を追加しましょう。

    ```Ruby
    platform = RUBY_PLATFORM.match(/(linux|darwin)/)[0].to_sym
    Bundler.require(platform)
    ```

* `Gemfile.lock` をバージョン管理下から外すのはやめましょう。
  これはランダムに生成されたファイルではありません。
  チームメンバー全員が、 `bundle install` をした場合に、
  確実に同じバージョンの gem を得ることができるようにするものです。

## Priceless Gems

最も重要なプログラミング原則の1つは "車輪の再発明をしない!!!" です。
もし、あなたが特定のタスクに直面している場合は、自分をアンロールする前に
既存のソリューションをちょっとばかし見回すべきです。
ここにいくつかの貴重な gem のリストがあります。
(全て Rails 3.1 に準拠しています) これらは、沢山の Rails プロジェクトで
有用です。

* [active_admin](https://github.com/gregbell/active_admin) - With ActiveAdmin
  the creation of admin interface for your Rails app is child's play. You get a
  nice dashboard, CRUD UI and lots more. Very flexible and customizable.
* [capybara](https://github.com/jnicklas/capybara) - Capybara aims to simplify
  the process of integration testing Rack applications, such as Rails, Sinatra
  or Merb. Capybara simulates how a real user would interact with a web
  application. It is agnostic about the driver running your tests and currently
  comes with Rack::Test and Selenium support built in. HtmlUnit, WebKit and
  env.js are supported through external gems. Works great in combination with
  RSpec & Cucumber.
* [carrierwave](https://github.com/jnicklas/carrierwave) - the ultimate file
  upload solution for Rails. Support both local and cloud storage for the
  uploaded files (and many other cool things). Integrates great with
  ImageMagick for image post-processing.
* [client_side_validations](https://github.com/bcardarella/client_side_validations) -
  Fantastic gem that automatically creates JavaScript client-side validations
  from your existing server-side model validations. Highly recommended!
* [compass-rails](https://github.com/chriseppstein/compass) - Great gem that
  adds support for some css frameworks. Includes collection of sass mixins that
  reduces code of css files and help fight with browser incompatibilities.
* [cucumber-rails](https://github.com/cucumber/cucumber-rails) - Cucumber is
  the premium tool to develop feature tests in Ruby. cucumber-rails provides
  Rails integration for Cucumber.
* [devise](https://github.com/plataformatec/devise) - Devise is full-featured
  authentication solution for Rails applications. In most cases it's preferable
  to use devise to unrolling your custom authentication solution.
* [fabrication](http://fabricationgem.org/) - a great fixture replacement
  (editor's choice).
* [factory_girl](https://github.com/thoughtbot/factory_girl) - an alternative
  to fabrication. Nice and mature fixture replacement. Spiritual ancestor of
  fabrication.
* [faker](http://faker.rubyforge.org/) - handy gem to generate dummy data
  (names, addresses, etc).
* [feedzirra](https://github.com/pauldix/feedzirra) - Very fast and flexible
  RSS/Atom feed parser.
* [friendly_id](https://github.com/norman/friendly_id) - Allows creation of
  human-readable URLs by using some descriptive attribute of the model instead
  of its id.
* [guard](https://github.com/guard/guard) - fantastic gem that monitors file
  changes and invokes tasks based on them. Loaded with lots of useful
  extension. Far superior to autotest and watchr.
* [haml-rails](https://github.com/indirect/haml-rails) - haml-rails provides
  Rails integration for Haml.
* [haml](http://haml-lang.com) - HAML is a concise templating language,
  considered by many (including yours truly) to be far superior to Erb.
* [kaminari](https://github.com/amatsuda/kaminari) - Great paginating solution.
* [machinist](https://github.com/notahat/machinist) - Fixtures aren't fun.
  Machinist is.
* [rspec-rails](https://github.com/rspec/rspec-rails) - RSpec is a replacement
  for Test::MiniTest. I cannot recommend highly enough RSpec. rspec-rails
  provides Rails integration for RSpec.
* [simple_form](https://github.com/plataformatec/simple_form) - once you've
  used simple_form (or formtastic) you'll never want to hear about Rails's
  default forms. It has a great DSL for building forms and no opinion on
  markup.
* [simplecov-rcov](https://github.com/fguillen/simplecov-rcov) - RCov formatter
  for SimpleCov. Useful if you're trying to use SimpleCov with the Hudson
  contininous integration server.
* [simplecov](https://github.com/colszowka/simplecov) - code coverage tool.
  Unlike RCov it's fully compatible with Ruby 1.9. Generates great reports.
  Must have!
* [slim](http://slim-lang.com) - Slim is a concise templating language,
  considered by many far superior to HAML (not to mention Erb). The only thing
  stopping me from using Slim massively is the lack of good support in major
  editors/IDEs. Its performance is phenomenal.
* [spork](https://github.com/timcharper/spork) - A DRb server for testing
  frameworks (RSpec / Cucumber currently) that forks before each run to ensure
  a clean testing state. Simply put it preloads a lot of test environment and
  as consequence the startup time of your tests in greatly decreased. Absolute
  must have!
* [sunspot](https://github.com/sunspot/sunspot) - SOLR powered full-text search
  engine.

  ※暇なときに翻訳しよう。

このリストは徹底的ではありませんし、他の gem が追加されるでしょう。(along the roadってどういう意味?)
リスト上の全ての gem はフィールドテストがされており、活発な開発とコミュニティがあり、
良いコード品質であることが知られています。

## Flawed Gems

これは、いずれかの問題であるか、他の gem で代替された gem のリストです。
プロジェクトでこれらを使うのは避けるべきでしょう。

* [rmagick](http://rmagick.rubyforge.org/) - this gem is notorious for its memory consumption. Use
[minimagick](https://github.com/probablycorey/mini_magick) instead.
* [autotest](http://www.zenspider.com/ZSS/Products/ZenTest/) - old solution for running tests automatically. Far
inferior to guard and [watchr](https://github.com/mynyml/watchr).
* [rcov](https://github.com/relevance/rcov) - code coverage tool, not
  compatible with Ruby 1.9. Use
  [SimpleCov](https://github.com/colszowka/simplecov) instead.
* [therubyracer](https://github.com/cowboyd/therubyracer) - the use of
  this gem in production is strongly discouraged as it uses a very large amount of
  memory. I'd suggest using
  [Mustang](https://github.com/nu7hatch/mustang) instead.

このリストも作成中です。良く知られているのだけれど、欠陥のある gem を
知っていたら、私に教えてください。

## Managing processes

* もし、プロジェクトがさまざまな外部プロセスに依存しているなら、
  [foreman](https://github.com/ddollar/foreman) を使って
  それらを管理しましょう。

# Testing Rails applications

新しい機能を実装する最善のアプローチは、おそらくBDDのアプローチです。
いくつかの高レベルの機能テストを書くことから(一般的に Cucumber を使って書かれた)始め、
その後、機能の実装をやっつけていくのにこれらのテストを使用します。
まず、機能のために view spec を書き、そして、関連するビューを作成するために
これらの spec を利用します。
その後は、View にデータを与えるコントローラーの spec を作成し、
コントローラーを実装するために spec を利用します。
最後に、モデルの spec とモデル自体を実装します。

## Cucumber

* 保留中のシナリオには `@wip` (作業中)タグをつけましょう。
  これらのシナリオは考慮されず、失敗としてマークされません。
  保留中のシナリオの作業が終わり、機能的な実装が終わったら、
  テストスイートにこのシナリオを含めるために `@wip` タグを削除すべきです。
* `@javascript` タグのついたシナリオを除外するデフォルトのプロファイルを
  用意しましょう。これらはブラウザを使ってテストされるので、
  無効化すると、定期的なシナリオの実行速度を向上させることができるでしょう。
* `@javascript` タグが付いているシナリオのためのプロファイルを別途用意しましょう。
  * プロファイルは `cucumber.yml` で設定することができます。

        ```Ruby
        # definition of a profile:
        profile_name: --tags @tag_name
        ```

  * プロファイルは以下のコマンドで実行できます。

        ```
        cucumber -p profile_name
        ```

* もし [fabrication](http://fabricationgem.org/) を fixtures の代替に
  利用している場合は、 事前に定義された
  [fabrication steps](http://fabricationgem.org/#!cucumber-steps) を使いましょう。
* 古い `web_steps.rb` を使ってはいけません!
  [The web steps were removed from the latest version of Cucumber.](http://aslakhellesoy.com/post/11055981222/the-training-wheels-came-off)
  これらは正しくアプリケーションドメインを反映していない冗長なシナリオの
  作成に繋がります。
* element や表示されているテキスト(リンク、ボタン など)のチェックをする場合は、
  テキストをチェックしましょう、element id をチェックしないように。
  これらは国際化の問題を検出することができます。
* 同じオブジェクトの種類に関するさまざまな機能ごとに、個別のフィーチャを作りましょう。


    ```Ruby
    # bad
    Feature: Articles
    # ... feature  implementation ...

    # good
    Feature: Article Editing
    # ... feature  implementation ...

    Feature: Article Publishing
    # ... feature  implementation ...

    Feature: Article Search
    # ... feature  implementation ...

    ```

* それぞれのフィーチャーには3つの主要なコンポーネントがあります。
  * タイトル
  * 物語 - 機能についての短い説明
  * 合否判定基準 - シナリオのセットは個々のステップで構成されています。
* 最も一般的なフォーマットは Connextra フォーマットとして知られています。

    ```Ruby
    In order to [benefit] ...
    A [stakeholder]...
    Wants to [feature] ...
    ```

この形式は最も一般的ですが、必須ではありません。
物語は機能の複雑さに応じて、テキストにすることができます。

* シナリオをDRYに保つために、Scenario Outline 使いましょう。

    ```Ruby
    Scenario Outline: User cannot register with invalid e-mail
      When I try to register with an email "<email>"
      Then I should see the error message "<error>"

    Examples:
      |email         |error                 |
      |              |The e-mail is required|
      |invalid email |is not a valid e-mail |
    ```

* シナリオのための step は `step_definitions` ディレクトリの
  `.rb` ファイルに含まれています。
  step ファイルの命名規則は `[description_steps.rb` です。
  step は異なる基準に基づいて、異なるファイルに分離することができます。
  それぞれの機能ごと (`home_page_steps.rb`) に1つの step ファイルを用意することもできます。
  また、特定のオブジェクト (`articles_steps.rb`) の全ての機能を1つの step ファイルにすることもできます。
* 繰り返しを避けるために、複数行の引数をもつステップを使いましょう。

    ```Ruby
    Scenario: User profile
      Given I am logged in as a user "John Doe" with an e-mail "user@test.com"
      When I go to my profile
      Then I should see the following information:
        |First name|John         |
        |Last name |Doe          |
        |E-mail    |user@test.com|

    # the step:
    Then /^I should see the following information:$/ do |table|
      table.raw.each do |field, value|
        find_field(field).value.should =~ /#{value}/
      end
    end
    ```

* シナリオをDRYに保つために、複合化した step を使いましょう。

    ```Ruby
    # ...
    When I subscribe for news from the category "Technical News"
    # ...

    # the step:
    When /^I subscribe for news from the category "([^"]*)"$/ do |category|
      steps %Q{
        When I go to the news categories page
        And I select the category #{category}
        And I click the button "Subscribe for this category"
        And I confirm the subscription
      }
    end
    ```

* 常に、should_not の代わりに Capybara negative matchers を使いましょう。
  (良く分からん)
  [See Capybara's README for more explanation](https://github.com/jnicklas/capybara)

## RSpec

  example ごとに1つの expectation を使いましょう。

    ```Ruby
    # bad
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns new article and renders the new article template' do
          get :new
          assigns[:article].should be_a_new Article
          response.should render_template :new
        end
      end

      # ...
    end

    # good
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns a new article' do
          get :new
          assigns[:article].should be_a_new Article
        end

        it 'renders the new article template' do
          get :new
          response.should render_template :new
        end
      end

    end
    ```

* `describe` と `context` を多用しましょう。
* `describe` ブロックを次のように名づけます。
  * メソッドでないもののために、 "description" を使う
  * インスタンスメソッドのために "#method" を使う
  * クラスメソッドのために ".method" を使う。

    ```Ruby
    class Article
      def summary
        #...
      end

      def self.latest
        #...
      end
    end

    # the spec...
    describe Article
      describe '#summary'
        #...
      end

      describe '.latest'
        #...
      end
    end
    ```

* テストオブジェクトを作成するために [fabricators](http://fabricationgem.org/)
  を使いましょう。
* mock と stab を使いまくりましょう。

    ```Ruby
    # mocking a model
    article = mock_model(Article)

    # stubbing a method
    Article.stub(:find).with(article.id).and_return(article)
    ```

* モデルをモックする場合は、 `as_nul_object` メソッドを使いましょう。
  これは 期待するメッセージのみをリスニングし、他のメッセージを無視するよう
  指示します。

    ```Ruby
    article = mock_model(Article).as_null_object
    ```

* spec example のためのデータを作成する場合は、
  `before(:all)` ブロックの代わりに、 `let` ブロックを使いましょう。
  `let` ブロックは遅延評価です。

    ```Ruby
    # use this:
    let(:article) { Fabricate(:article) }

    # ... instead of this:
    before(:each) { @article = Fabricate(:article) }
    ```

* 可能であれば、 `subject` を使いましょう。

    ```Ruby
    describe Article do
      subject { Fabricate(:article) }

      it 'is not published on creation' do
        subject.should_not be_published
      end
    end
    ```

* 可能であれば、 `specify` を使いましょう。
  これは、 `it` と同じ意味ですが、docstring がない場合に、より読みやすくなります。

    ```Ruby
    # bad
    describe Article do
      before { @article = Fabricate(:article) }

      it 'is not published on creation' do
        @article.should_not be_published
      end
    end

    # good
    describe Article do
      let(:article) { Fabricate(:article) }
      specify { article.should_not be_published }
    end
    ```

* 可能であれば、 `its` を使いましょう。

    ```Ruby
    # bad
    describe Article do
      subject { Fabricate(:article) }

      it 'has the current date as creation date' do
        subject.creation_date.should == Date.today
      end
    end

    # good
    describe Article do
      subject { Fabricate(:article) }
      its(:creation_date) { should == Date.today }
    end
    ```

### Views

* View spec `spec/views` のディレクトリ構造は、 `app/views` の
  1つに一致します。
  たとえば、 `app/views/users` の spec は `spec/views/users` に配置されます。
* view spec の命名規則は `_spec.rb` に view の名前を追加します。
  たとえば、 `_form.html.haml` に対応する spec は `_form.html.haml_spec.rb` です。
* `spec_helper.rb` はそれぞれの spec ファイルで require する必要があります。
* 外部の `describe` ブロックは、 `app/views` 部分を除いた view のパスを使います。
  このパスは、 引数なしで呼び出された `render` メソッドが使うものです。
 
    ```Ruby
    # spec/views/articles/new.html.haml_spec.rb
    require 'spec_helper'

    describe 'articles/new.html.haml' do
      # ...
    end
    ```

* view spec では常にモデルはモックにしましょう。
  view の目的は情報を表示するのみです。
* `assign` メソッドはインスタンス変数を提供します。
  これは view が使用するもので、コントローラーが提供します。

    ```Ruby
    # spec/views/articles/edit.html.haml_spec.rb
    describe 'articles/edit.html.haml' do
    it 'renders the form for a new article creation' do
      assign(
        :article,
        mock_model(Article).as_new_record.as_null_object
      )
      render
      rendered.should have_selector('form',
        method: 'post',
        action: articles_path
      ) do |form|
        form.should have_selector('input', type: 'submit')
      end
    end
    ```

* should_not より、Capybara の negative セレクタを使いましょう。

    ```Ruby
    # bad
    page.should_not have_selector('input', type: 'submit')
    page.should_not have_xpath('tr')

    # good
    page.should have_no_selector('input', type: 'submit')
    page.should have_no_xpath('tr')
    ```

* View が helper メソッドを使う場合、これらのメソッドは
  スタブになっている必要があります。
  helper メソッドのスタブ化は、 `template` オブジェクト上で
  行われます。

    ```Ruby
    # app/helpers/articles_helper.rb
    class ArticlesHelper
      def formatted_date(date)
        # ...
      end
    end

    # app/views/articles/show.html.haml
    = "Published at: #{formatted_date(@article.published_at)}"

    # spec/views/articles/show.html.haml_spec.rb
    describe 'articles/show.html.html' do
      it 'displays the formatted date of article publishing'
        article = mock_model(Article, published_at: Date.new(2012, 01, 01))
        assign(:article, article)

        template.stub(:formatted_date).with(article.published_at).and_return '01.01.2012'

        render
        rendered.should have_content('Published at: 01.01.2012')
      end
    end
    ```

* helper の spec は `spec/helpers` ディレクトリに入れて、
  view の spec から分離しましょう。

### Controllers

* モデルをモック、そして、それらのメソッドをスタブ化しましょう。
  コントローラのテストは、モデルの作成に依存すべきではありません。
* コントローラが責任を負うべき振る舞いだけをテストしましょう。
  * 特定のメソッドの実行
  * アクションから返されたデータ - 割り当てなど
  * アクションの結果 - テンプレートのレンダリング、リダイレクトなど

        ```Ruby
        # Example of a commonly used controller spec
        # spec/controllers/articles_controller_spec.rb
        # We are interested only in the actions the controller should perform
        # So we are mocking the model creation and stubbing its methods
        # And we concentrate only on the things the controller should do

        describe ArticlesController do
          # The model will be used in the specs for all methods of the controller
          let(:article) { mock_model(Article) }

          describe 'POST create' do
            before { Article.stub(:new).and_return(article) }

            it 'creates a new article with the given attributes' do
              Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
              post :create, message: { title: 'The New Article Title' }
            end

            it 'saves the article' do
              article.should_receive(:save)
              post :create
            end

            it 'redirects to the Articles index' do
              article.stub(:save)
              post :create
              response.should redirect_to(action: 'index')
            end
          end
        end
        ```

* コントローラのアクションが、受け取ったパラメータに依存して異なる振る舞いを
  する場合は、 context を使いましょう。

    ```Ruby
    # A classic example for use of contexts in a controller spec is creation or update when the object saves successfully or not.

    describe ArticlesController do
      let(:article) { mock_model(Article) }

      describe 'POST create' do
        before { Article.stub(:new).and_return(article) }

        it 'creates a new article with the given attributes' do
          Article.should_receive(:new).with(title: 'The New Article Title').and_return(article)
          post :create, article: { title: 'The New Article Title' }
        end

        it 'saves the article' do
          article.should_receive(:save)
          post :create
        end

        context 'when the article saves successfully' do
          before { article.stub(:save).and_return(true) }

          it 'sets a flash[:notice] message' do
            post :create
            flash[:notice].should eq('The article was saved successfully.')
          end

          it 'redirects to the Articles index' do
            post :create
            response.should redirect_to(action: 'index')
          end
        end

        context 'when the article fails to save' do
          before { article.stub(:save).and_return(false) }

          it 'assigns @article' do
            post :create
            assigns[:article].should be_eql(article)
          end

          it 're-renders the "new" template' do
            post :create
            response.should render_template('new')
          end
        end
      end
    end
    ```

### Models

* モデル自身の spec で、モデルをモックにしないように。
* 本当のオブジェクトを作成するために、 fabrication を使いましょう。
* 他のモデルや、子供のオブジェクトをモックにするのは許容できます。
* 重複を避けるために、 spec の全ての example のためにモデルを作成しましょう。

    ```Ruby
    describe Article
      let(:article) { Fabricate(:article) }
    end
    ```

* fabricated モデルが valid であることを保証するための example を追加しましょう。

    ```Ruby
    describe Article
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* validation をテストする場合は、 validate されるべき attribute を特定するために、
  `have(x).errors_on` を使いましょう。
  `be_valid` を使うと、問題が意図した attribute であるかどうかを保証できません。
  
    ```Ruby
    # bad
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should_not be_valid
      end
    end

    # prefered
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

* validation を持つ attribute ごとに `describe` を分けましょう。

    ```Ruby
    describe Article
      describe '#title'
        it 'is required' do
          article.title = nil
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

* モデルの attribute の唯一性をテストする場合は、他のオブジェクトに
  `another_object` という名前をつけましょう。

    ```Ruby
    describe Article
      describe '#title'
        it 'is unique' do
          another_article = Fabricate.build(:article, title: article.title)
          article.should have(1).error_on(:title)
        end
      end
    end
    ```

### Mailers

* メーラー spec にあるモデルはモックにするべきです。
  メーラーはモデルの生成に依存してはいけません。
* メーラー spec では以下を確認します。
  * 件名が正しい
  * 受信者の電子メールが正しい
  * 正しい電子メールアドレスに送信されている
  * 電子メールに必要な情報が含まれている

     ```Ruby
     describe SubscriberMailer
       let(:subscriber) { mock_model(Subscription, email: 'johndoe@test.com', name: 'John Doe') }

       describe 'successful registration email'
         subject { SubscriptionMailer.successful_registration_email(subscriber) }

         its(:subject) { should == 'Successful Registration!' }
         its(:from) { should == ['info@your_site.com'] }
         its(:to) { should == [subscriber.email] }

         it 'contains the subscriber name' do
           subject.body.encoded.should match(subscriber.name)
         end
       end
     end
     ```

### Uploaders

* アップローダに対して私たちができるテストは、画像が正しくリサイズされているかどうか  です。
  ここに画像アップローダー [carrierwave](https://github.com/jnicklas/carrierwave) のサンプル spec があります。

    ```Ruby

    # rspec/uploaders/person_avatar_uploader_spec.rb
    require 'spec_helper'
    require 'carrierwave/test/matchers'

    describe PersonAvatarUploader do
      include CarrierWave::Test::Matchers

      # Enable images processing before executing the examples
      before(:all) do
        UserAvatarUploader.enable_processing = true
      end

      # Create a new uploader. The model is mocked as the uploading and resizing images does not depend on the model creation.
      before(:each) do
        @uploader = PersonAvatarUploader.new(mock_model(Person).as_null_object)
        @uploader.store!(File.open(path_to_file))
      end

      # Disable images processing after executing the examples
      after(:all) do
        UserAvatarUploader.enable_processing = false
      end

      # Testing whether image is no larger than given dimensions
      context 'the default version' do
        it 'scales down an image to be no larger than 256 by 256 pixels' do
          @uploader.should be_no_larger_than(256, 256)
        end
      end

      # Testing whether image has the exact dimensions
      context 'the thumb version' do
        it 'scales down an image to be exactly 64 by 64 pixels' do
          @uploader.thumb.should have_dimensions(64, 64)
        end
      end
    end

    ```

# 参考文献

時間に余裕があるときに、みるべき Rails style に関するいくつかの優れたリソースがあります。

* [The Rails 3 Way](http://tr3w.com/)
* [Ruby on Rails Guides](http://guides.rubyonrails.org/)
* [The RSpec Book](http://pragprog.com/book/achbd/the-rspec-book)

# Contributing

このガイドでかかれたものは、不変ではありません。
我々が最終的に全体のRubyコミュニティにとって有益なリソースを作成することができるように、
みんながRailsのコーディングスタイルに興味を持って、一緒に作り上げていくことが私の願望です。

改善のためのチケット、プルリクエストの送信はご自由に。
あなたの助けに感謝します。

# 世界を広げる

コミュニティ主導のスタイルガイドではその存在を知らないコミュニティにはほとんど役に立ちません。
ガイドに関する Tweet は、あなたの友人や同僚と共有することができます。
我々が得るすべてのコメント、提案または意見はほんの少し良いガイドになります。
可能な限り最高のガイドを持ちたいと思いませんか？
