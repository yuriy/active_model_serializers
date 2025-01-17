[Back to Guides](../README.md)

# Serializers

Given a serializer class:

```ruby
class SomeSerializer < ActiveModel::Serializer
end
```

The following methods may be defined in it:

### Attributes

#### ::attributes

Serialization of the resource `title` and `body`

| In Serializer               | #attributes |
|---------------------------- |-------------|
| `attributes :title, :body`  | `{ title: 'Some Title', body: 'Some Body' }`
| `attributes :title, :body`<br>`def body "Special #{object.body}" end` | `{ title: 'Some Title', body: 'Special Some Body' }`


#### ::attribute

Serialization of the resource `title`

| In Serializer               | #attributes |
|---------------------------- |-------------|
| `attribute :title`          | `{ title: 'Some Title' } `
| `attribute :title, key: :name` | `{ name: 'Some Title' } `
| `attribute :title { 'A Different Title'}` | `{ title: 'A Different Title' } `
| `attribute :title`<br>`def title 'A Different Title' end` | `{ title: 'A Different Title' }`

[PR please for conditional attributes:)](https://github.com/rails-api/active_model_serializers/pull/1403)

### Associations

#### ::has_one

e.g.

```ruby
has_one :bio
has_one :blog, key: :site
has_one :maker, virtual_value: { id: 1 }
```

#### ::has_many

e.g.

```ruby
has_many :comments
has_many :comments, key: :reviews
has_many :comments, serializer: CommentPreviewSerializer
has_many :reviews, virtual_value: [{ id: 1 }, { id: 2 }]
has_many :comments, key: :last_comments do
  last(1)
end
```

#### ::belongs_to

e.g.

```ruby
belongs_to :author, serializer: AuthorPreviewSerializer
belongs_to :author, key: :writer
belongs_to :post
belongs_to :blog
def blog
  Blog.new(id: 999, name: 'Custom blog')
end
```

### Caching

#### ::cache

e.g.

```ruby
cache key: 'post', expires_in: 0.1, skip_digest: true
cache expires_in: 1.day, skip_digest: true
cache key: 'writer', skip_digest: true
cache only: [:name], skip_digest: true
cache except: [:content], skip_digest: true
cache key: 'blog'
cache only: [:id]
```

#### #cache_key

e.g.

```ruby
# Uses a custom non-time-based cache key
def cache_key
  "#{self.class.name.downcase}/#{self.id}"
end
```

### Other

#### ::type

The `::type` method defines the JSONAPI [type](http://jsonapi.org/format/#document-resource-object-identification) that will be rendered for this serializer.
It either takes a `String` or `Symbol` as parameter.

Note: This method is useful only when using the `:json_api` adapter.

Examples:
```ruby
class UserProfileSerializer < ActiveModel::Serializer
  type 'profile'
end
class AuthorProfileSerializer < ActiveModel::Serializer
  type :profile
end
```

With the `:json_api` adapter, the previous serializers would be rendered as:

``` json
{
  "data": {
    "id": "1",
    "type": "profile"
  }
}
```

#### ::link

```ruby
link :self do
  href "https://example.com/link_author/#{object.id}"
end
link :author { link_author_url(object) }
link :link_authors { link_authors_url }
link :other, 'https://example.com/resource'
link :posts { link_author_posts_url(object) }
```

#### #object

The object being serialized.

#### #root

PR please :)

#### #scope

Allows you to include in the serializer access to an external method.

It's intended to provide an authorization context to the serializer, so that
you may e.g. show an admin all comments on a post, else only published comments.

- `scope` is a method on the serializer instance that comes from `options[:scope]`. It may be nil.
- `scope_name` is an option passed to the new serializer (`options[:scope_name]`).  The serializer
  defines a method with that name that calls the `scope`, e.g. `def current_user; scope; end`.
  Note: it does not define the method if the serializer instance responds to it.

That's a lot of words, so here's some examples:

First, let's assume the serializer is instantiated in the controller, since that's the usual scenario.
We'll refer to the serialization context as `controller`.

| options | `Serializer#scope` | method definition |
|-------- | ------------------|--------------------|
| `scope: current_user, scope_name: :current_user` | `current_user` | `Serializer#current_user` calls `controller.current_user`
| `scope: view_context, scope_name: :view_context` | `view_context` | `Serializer#view_context` calls `controller.view_context`

We can take advantage of the scope to customize the objects returned based
on the current user (scope).

For example, we can limit the posts the current user sees to those they created:

```ruby
class PostSerializer < ActiveModel::Serializer
  attributes :id, :title, :body

  # scope comments to those created_by the current user
  has_many :comments do
    object.comments.where(created_by: current_user)
  end
end
```

Whether you write the method as above or as `object.comments.where(created_by: scope)`
is a matter of preference (assuming `scope_name` has been set).

##### Controller Authorization Context

In the controller, the scope/scope_name options are equal to
the [`serialization_scope`method](https://github.com/rails-api/active_model_serializers/blob/d02cd30fe55a3ea85e1d351b6e039620903c1871/lib/action_controller/serialization.rb#L13-L20),
which is `:current_user`, by default.

Specfically, the `scope_name` is defaulted to `:current_user`, and may be set as
`serialization_scope :view_context`.  The `scope` is set to `send(scope_name)` when `scope_name` is
present and the controller responds to `scope_name`.

Thus, in a serializer, the controller provides `current_user` as the
current authorization scope when you call `render :json`.

**IMPORTANT**: Since the scope is set at render, you may want to customize it so that `current_user` isn't
called on every request.  This was [also a problem](https://github.com/rails-api/active_model_serializers/pull/1252#issuecomment-159810477)
in [`0.9`](https://github.com/rails-api/active_model_serializers/tree/0-9-stable#customizing-scope).

We can change the scope from `current_user` to `view_context`.

```diff
class SomeController < ActionController::Base
+  serialization_scope :view_context

  def current_user
    User.new(id: 2, name: 'Bob', admin: true)
  end

  def edit
    user = User.new(id: 1, name: 'Pete')
    render json: user, serializer: AdminUserSerializer, adapter: :json_api
  end
end
```

We could then use the controller method `view_context` in our serializer, like so:

```diff
class AdminUserSerializer < ActiveModel::Serializer
  attributes :id, :name, :can_edit

  def can_edit?
+    view_context.current_user.admin?
  end
end
```

So that when we render the `#edit` action, we'll get

```json
{"data":{"id":"1","type":"users","attributes":{"name":"Pete","can_edit":true}}}
```

Where `can_edit` is `view_context.current_user.admin?` (true).

#### #read_attribute_for_serialization(key)

The serialized value for a given key. e.g. `read_attribute_for_serialization(:title) #=> 'Hello World'`

#### #links

PR please :)

#### #json_key

PR please :)

## Examples

Given two models, a `Post(title: string, body: text)` and a
`Comment(name: string, body: text, post_id: integer)`, you will have two
serializers:

```ruby
class PostSerializer < ActiveModel::Serializer
  cache key: 'posts', expires_in: 3.hours
  attributes :title, :body

  has_many :comments
end
```

and

```ruby
class CommentSerializer < ActiveModel::Serializer
  attributes :name, :body

  belongs_to :post
end
```

Generally speaking, you, as a user of ActiveModelSerializers, will write (or generate) these
serializer classes.

## More Info

For more information, see [the Serializer class on GitHub](https://github.com/rails-api/active_model_serializers/blob/master/lib/active_model/serializer.rb)

## Overriding association methods

To override an association, call `has_many`, `has_one` or `belongs_to` with a block:

```ruby
class PostSerializer < ActiveModel::Serializer
  has_many :comments do
    object.comments.active
  end
end
```

## Overriding attribute methods

To override an attribute, call `attribute` with a block:

```ruby
class PostSerializer < ActiveModel::Serializer
  attribute :body do
    object.body.downcase
  end
end
```
