# CashIO Backend

CashIO API built with Ruby on Rails.

[![Ruby Style Guide](https://img.shields.io/badge/code_style-rubocop-brightgreen.svg)](https://github.com/rubocop/rubocop)

## Useful Commands

### Create new model:

```
$ rails g model Entry name:string phone:string last_purchase:date
```

### Create new database based on app/models/\*.rb:

```
$ rails db:create
```

### Migrate created database:

```
$ rails db:migrate
```

### Prepare tests for created database:

```
$ rails db:test:prepare
```

### Populate database based on db/seeds.rb:

```
$ rails db:seed RAILS_ENV=development
```

### Generate controller with CRUD operations:

```
$ rails g scaffold_controller Entry
```

### Generate Kaminari gem config file, to implement pagination:

```
$ rails g kaminari:config
```

### Generate uploader:

```
$ rails g uploader Avatar
```

### Show all api routes:

```
$ rails routes
```

### Serve app:

```
$ rails s
```

## Useful gems

`gem 'rack-cors'` => Makes cross-origin AJAX possible.

`gem 'kaminari'` => Pagination.

`gem 'pg_search'` => PostgreSQL search.

`gem 'carrierwave', '>= 3.0.0.beta', '< 4.0'` => File upload.

`gem 'rubocop-rails', require: false` => Good pratices and code style.

`gem 'dotenv-rails', groups: %i[development test]'` => Enable .env files.

## Code Examples

### Database config file with PostgreSQL setup:

#### ./config/database.yml

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: postgres
  password: root

development:
  <<: *default
  database: cash_io_development

test:
  <<: *default
  database: cash_io_test

production:
  <<: *default
  database: cash_io_production
  username: cash_io
  password: <%= ENV["CASH_IO_DATABASE_PASSWORD"] %>
```

### Gemfile:

### ./Gemfile

```rb
source 'https://rubygems.org'
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby '3.1.3'

gem 'bcrypt', '~> 3.1.7'
gem 'bootsnap', require: false
gem 'carrierwave', '>= 3.0.0.beta', '< 4.0'
gem 'dotenv-rails', groups: %i[development test]
gem 'jwt'
gem 'kaminari'
gem 'pg', '~> 1.1'
gem 'pg_search'
gem 'puma', '~> 5.0'
gem 'rack-cors'
gem 'rails', '~> 7.0.4', '>= 7.0.4.3'
gem 'tzinfo-data', platforms: %i[mingw mswin x64_mingw jruby]

group :development, :test do
  gem 'debug', platforms: %i[mri mingw x64_mingw]
end

group :development do
  gem 'faker'
  gem 'rubocop-rails', require: false
end
```

### Routes:

#### ./config/routes.rb

```rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :entries
    end
  end
end
```

### Entity Model with pg_search gem setup and validation:

#### ./app/models/entry.rb

```rb
class Entry < ApplicationRecord
  include PgSearch::Model

  validates :name, presence: true
  validates :date, presence: true
  validates :value, presence: true

  pg_search_scope :search_by_term,
    against: :name,
    using: {
      tsearch: {
        any_word: true,
        prefix: true,
      },
    }
end
```

### CRUD controller with pagination, order by, direction and search:

#### ./app/controllers/api/v1/entries_controller.rb

```rb
class Api::V1::EntriesController < ApplicationController
  before_action :set_entry, only: %i[ show update destroy ]

  # GET /entries
  def index
    @direction = "ASC"
    if params[:direction]
      @direction = params[:direction]
    end

    @order_by = "id"
    if params[:order_by]
      @order_by = params[:order_by]
    end

    @page = 1
    if params[:page]
      @page = params[:page].to_i
    end

    @per_page = 25
    if params[:per_page]
      @per_page = params[:per_page].to_i
    end

    @search = params[:search]

    @result = Entry.order("#@order_by #@direction").page(@page).per(@per_page)

    if @search
      @result = @result.search_by_term(@search)
    end

    @total = @result.total_count

    @last_page = @total.fdiv(@per_page).ceil

    render json: {
      result: @result,
      direction: @direction,
      order_by: @order_by,
      page: @page,
      per_page: @per_page,
      search: @search,
      total: @total,
      last_page: @last_page,
    }
  end

  # GET /entries/1
  def show
    render json: @entry
  end

  # POST /entries
  def create
    @entry = Entry.new(entry_params)

    if @entry.save
      render json: @entry, status: :created
    else
      render json: @entry.errors, status: :unprocessable_entity
    end
  end

  # PATCH/PUT /entries/1
  def update
    if @entry.update(entry_params)
      render json: @entry
    else
      render json: @entry.errors, status: :unprocessable_entity
    end
  end

  # DELETE /entries/1
  def destroy
    @entry.destroy
  end

  private

  # Use callbacks to share common setup or constraints between actions.
  def set_entry
    @entry = Entry.find(params[:id])
  end

  # Only allow a list of trusted parameters through.
  def entry_params
    params.require(:entry).permit(:name, :description, :date, :value)
  end
end
```

### Seeds:

#### ./db/seeds.rb

```rb
50.times do
  Entry.create({
    name: Faker::Name.name,
    description: Faker::Lorem.paragraph,
    date: Faker::Date.between(from: 30.days.ago, to: Date.today),
    value: Faker::Number.between(from: -1000.0, to: 1000.0),
  })
end
```
