# Pokedex: Part 2 - The Backend

In today's project, you will create a lean Rails backend for the Pokedex
frontend that you created yesterday. Along the way, you will learn some new
syntactic sugar, investigate how Rails handles parameters, and practice building
Jbuilder templates.

## Phase 1 - Create a New Rails Project and Set Up the Database

To begin, create a new Rails project:

```shell
rails new pokedex-rails-backend -G --database=postgresql --api --minimal
```

To specify a particular Rails version, include the version number right after
`rails`, e.g., `rails _6.1.4_ new ....`.

Recall that the `-G` flag keeps Rails from installing a new git repo in the
project; if you want the new git repo installed--make sure it won't create
nested repos!--you can omit the `-G`. (You can grab an appropriate
__.gitignore__ file from the repo linked to the `Download Project` button at the
bottom of this page.)

The `--database` (or `-d`) flag tells Rails to use PostgreSQL as the database.

The `--api` flag tells Rails that you will be using this application primarily
as a backend API, so it will not install support for features such as Views--no
asset pipeline!--and Session. For more information on the `--api` flag and what
it includes/omits, see the [docs][rails-api].

Finally, `--minimal` instructs Rails to build a minimalist application: it will
not install `ActionCable`, `ActionMailbox`, `ActionMailer`, `ActiveJob`,
`ActiveStorage`, and so on. These are wonderful features all, but you will not
need them for this project. Anything you do need (e.g., Jbuilder), you can just
add back in.

Next, add the following gems to your Gemfile:

* At the top level
  * `jbuilder`

* Under `group :development, :test`
  * `pry-rails`
  * `faker`

* Under `group :development`
  * `annotate`
  * `better_errors`
  * `binding_of_caller`

Run `bundle install`.

### Setting up the Database

To set up your database, make sure Postgres is running, then run `rails
db:create`. Now you need to fill the database with the appropriate tables. You
will need 4: `pokemons`, `items`, `moves`, and `poke_moves`. Run the following
command to generate a migration for the `pokemons` table and model:

```shell
rails g model Pokemon number:integer:uniq name:string:uniq attack:integer defense:integer poke_type:string image_url:string captured:boolean --no-test-framework
```

(The `--no-test-framework` flag prevents Rails from generating boilerplate test
files for your model that you are not going to use in this project. Omitting the
flag will not create any problems for your code.)

Take a look at the migration file that this command creates in __db/migrate__.
Notice that appending `:uniq` to `number:integer` and `name:string` effectively
added indexes for `number` and `name` with uniqueness constraints. Note, too,
that Rails has automatically added `t.timestamps`.

Go ahead and add `null: false` constraints to the other fields (`number`,
`name`, `attack`, `defense`, `poke_type`, `image_url`, and `captured`). Set
`captured` to default to `false` as well. Note that while setting a default
value on `captured` will assure that the field is not `null` in most cases, it
is still useful to have a `null: false` constraint to protect against instances
where `captured` is explicitly set to `null`. This could happen, for instance,
if an attempt to set `captured` dynamically fails.

Next create a migration for your `items` table and model:

```sh
rails g model Item pokemon:references name:string price:integer happiness:integer image_url:string --no-test-framework
```

Open the newly created migration file. Go ahead and add `null: false`
constraints to the `name`, `price`, `happiness`, and `image_url` fields. Note
that the `references` type is used to create foreign keys. It automatically adds
`null: false` and `foreign key` constraints. When you run the migration, the
`references` type will create a `pokemon_id` field (N.B.: **NOT** `pokemon`)
**and add an index for this field**.

The `foreign key` constraint prevents your app from deleting records needed by
other tables. E.g., running `Pokemon.destroy_all` when there are still valid
entries in the `items` table will produce an error like this:

```sh
PG::ForeignKeyViolation: ERROR:  update or delete on table "pokemons" violates foreign key constraint "fk_rails_e992d4d99e" on table "items"
```

To prevent such an error, you must ensure that deletions always occur in the
proper order. In the example above, for instance, you could simply run
`Item.destroy_all` before `Pokemon.destroy_all`. While you could always execute
these extra deletions manually, Rails provides a way to ensure that deleting a
Pokemon will automatically 1) delete such dependencies 2) in the proper order:
you simply add `dependent: :destroy` to the `has_many` relationship in the
model. (You will implement this option below.)

Finish creating your migrations by generating migrations for the `moves` and
`poke_moves` models. `moves` needs only a unique string `name`. Don't forget to
add a `null: false` constraint in the migration file!

`poke_moves` is a join table connecting a `move` with a `pokemon`. Create this
migration on the command line using the `references` type, then add a constraint
to ensure that each move associated with a given Pokemon is unique. To do this,
go into the migration file and add a composite index on `pokemon_id` and
`move_id` with a uniqueness constraint.

> **Note**: When creating the index, order matters. You should specify
> `pokemon_id` before `move_id` because your app will need to generate the moves
> for a given Pokemon far more often than it will need to generate the Pokemon
> associated with a given move.

Since the creation of the composite index will effectively create an index for
the first column--here, `pokemon_id`--you can also add `index: false` to the
`t.references :pokemon` line. This will keep your app from creating two distinct
indexes for `pokemon_id`.

Your `create_table` for `poke_moves` should now look something like this:

```rb
create_table :poke_moves do |t|
  t.references :pokemon, null: false, foreign_key: true, index: false
  t.references :move, null: false, foreign_key: true
  t.index [:pokemon_id, :move_id], unique: true

  t.timestamps
end
```

(**N.B.**: `poke_moves` requires that both the `pokemons` and the `moves` tables
already exist in order to create the foreign keys. This migration must
accordingly run **after** the migrations for those two tables or it will fail.)

Once you have finished creating your four migrations, run `rails
db:migrate`. The resulting __db/schema.rb__ file should look like this:

```rb
ActiveRecord::Schema.define(version: 2021_10_07_223542) do

  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"

  create_table "items", force: :cascade do |t|
    t.bigint "pokemon_id", null: false
    t.string "name", null: false
    t.integer "price", null: false
    t.integer "happiness", null: false
    t.string "image_url", null: false
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
    t.index ["pokemon_id"], name: "index_items_on_pokemon_id"
  end

  create_table "moves", force: :cascade do |t|
    t.string "name", null: false
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
    t.index ["name"], name: "index_moves_on_name", unique: true
  end

  create_table "poke_moves", force: :cascade do |t|
    t.bigint "pokemon_id", null: false
    t.bigint "move_id", null: false
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
    t.index ["move_id"], name: "index_poke_moves_on_move_id"
    t.index ["pokemon_id", "move_id"], name: "index_poke_moves_on_pokemon_id_and_move_id", unique: true
  end

  create_table "pokemons", force: :cascade do |t|
    t.integer "number", null: false
    t.string "name", null: false
    t.integer "attack", null: false
    t.integer "defense", null: false
    t.string "poke_type", null: false
    t.string "image_url", null: false
    t.boolean "captured", default: false, null: false
    t.datetime "created_at", precision: 6, null: false
    t.datetime "updated_at", precision: 6, null: false
    t.index ["name"], name: "index_pokemons_on_name", unique: true
    t.index ["number"], name: "index_pokemons_on_number", unique: true
  end

  add_foreign_key "items", "pokemons"
  add_foreign_key "poke_moves", "moves"
  add_foreign_key "poke_moves", "pokemons"
end
```

If anything went awry, create a migration to fix it.

### Validations and Associations

Look at the files in __app/models__; there should be one for each of the 4
models. Run `annotate --models` to import the relevant schema into
each of the model files. Add validations to each of your models, beginning with
__pokemon.rb__.

First, create the appropriate checks for `presence`. (Refer to the schema at the
top of the file!) Don't add validations for `created_at` or `updated_at`; Rails
will take care of validating and populating these two columns.

Did you add `captured` to the list of columns to check for `presence`? If so,
the validation will fail whenever `captured` is `false`, which is not the
desired behavior. Remember that for booleans, you have to validate `presence` by
checking that the value is either `true` or `false`. You can do this with an
`inclusion` validation:

```rb
# app/models/pokemon.rb
class Pokemon < ApplicationRecord
  # ...
  validates :captured, inclusion: [true, false]
  # ...
end
```

Next, add `length` and `uniqueness` validations for `name`. The `length` should
be between 3 and 255 characters, inclusive. (If you've forgotten how to validate
length, see the [Rails Validation Guide][length-validation].)

For `uniqueness`, include a custom error message that specifies the non-unique
`name`, stating that it is already in use. To do this, instead of specifying
`uniqueness: true`, make the value of `uniqueness` itself a hash with a key of
`message` and a value that is the string with your desired error message. Note
that you can dynamically access the value, attribute, and model for use in your
error message by using %{value}, %{attribute}, and %{model}, respectively. Here,
you can use %{value} to reference the particular non-unique `name`. (For more on
customizing error messages, see the [Guide][custom-error-messages].)

Your `name` validation should now look something like this:

```rb
# app/models/pokemon.rb
class Pokemon < ApplicationRecord
  # ...
  validates :name, length: { in: 3..255 }, uniqueness: { message: "'%{value}' is already in use" }
  # ...
end
```

Add a similar `uniqueness` validation with custom error message for `number`.

For `number`, `attack`, and `defense`, use the `numericality` validation to
ensure that all three fields fall within appropriate minimum and maximum values:
0-100 for `attack` and `defense`, > 0 for `number`. For help with `numericality`
validations, consult the [Guide][numericality].

Finally, validate that `poke_type` is an allowable type. To do this, declare an
array containing strings of the valid `poke_type`s and check for `inclusion`
within it. Add a custom error message specifying the invalid `poke_type` by name
as well:

```rb
# app/models/pokemon.rb
class Pokemon < ApplicationRecord
  TYPES = [
    'fire',
    'electric',
    'normal',
    'ghost',
    'psychic',
    'water',
    'bug',
    'dragon',
    'grass',
    'fighting',
    'ice',
    'flying',
    'poison',
    'ground',
    'rock',
    'steel'
  ].sort.freeze

  # ...
  validates :poke_type, inclusion: { in: TYPES, message: "'%{value}' is not a valid Pokemon type" }
  # ...
end
```

If you think about it, the validations for `inclusion`, `length`, and
`numericality` that you have added will all fail for a column that is blank,
i.e., a column whose value is `nil`. You can accordingly refactor your code to
eliminate the `presence` validation for any column with one of these additional
checks. In fact, the only column whose `presence` still needs validating is
`image_url`.

Now go through and add appropriate validations for `Items`, `Moves`, and
`PokeMoves`. Ensure that all `name`s are < 255 characters and that `price` in
`Item` is >= 0. If `name` in `Move` is not unique, provide a custom error
message specifying that the non-unique name is already in use. As for
`PokeMove`, Rails will automatically validate the `presence` of `belongs_to`
associations, so there's no need to add any additional `presence` validations.
You should, however, add a `uniqueness` validation to ensure that no `pokemon`
has the same `move` more than once. You will want to use the [:scope
option][scope] with a custom error message noting that "pokemon cannot have the
same move more than once".

As you added your validations, you probably noticed that Rails has already
supplied the `belongs_to` associations for each of the foreign keys. This is a
nice benefit of using the model generator with the `references` type! For each
`belongs_to`, now add a corresponding `has_many` in the appropriate model.
Remember to add the `dependent: :destroy` option where warranted. Add an
additional `has_many :moves` in the `Pokemon` model that goes through
`poke_moves`. Add the corresponding `has_many :pokemon` in `Move`, too. (For
`has_many :through` associations, see [here][has-many-through].)

### Seed and Test

Download the __seeds.rb__ file available from the repo linked to the `Download
Project` button at the bottom of this page. Replace __db/seeds.rb__ in your
project with this file. While you are copying files, go ahead and copy the
__images__ folder from that repo into your project's __/public__ folder.

Run `rails db:seed`, then start the Rails console (`rails c`). Make sure your
seeding worked by running `Pokemon.all` from within the console. You should see
a long list of Pokemon.

Next test your associations. Run `Pokemon.first.moves`. It should produce the
`tackle` and `vine whip` `Move`s. Then test that `Pokemon.first.items` produces
3 `Item`s, all with a `pokemon_id` of 1. If either of these commands produces
different results, check the `belongs_to` and `has_many` associations in your
model files. Finally, test that `Move.first.pokemon` returns the `Pokemon` with
`id`s 1, 2, and 3.

Now test your validations. Type `p = Pokemon.new(captured: nil)` to create a
`Pokemon` where every value is nil, then try `p.save!`. (Be sure to add the `!`
so you can see the error messages!) You should get an error similar to the
following (if you don't see all of these failures, go back and check the
validations in __app/models/pokemon.rb__):

```text
ActiveRecord::RecordInvalid: Validation failed: Image url can't be blank, Captured is not included in the list, Name is too short (minimum is 3 characters), Attack is not a number, Defense is not a number, Number is not a number, Poke type '' is not a valid Pokemon type
```

Note that the message for `captured` is not very clear/helpful; go back and set
a custom error message for that validation too. Continue to test your
validations, making sure to trigger each condition at least once. For instance,
try to `save!` this Pokemon:

```sh
p = Pokemon.new(number: 0, name: Pokemon.first.name, attack: -1, defense: 101, poke_type: 'space', image_url: '1.svg', captured: nil)
```

This time, you should see an error along the following lines:

```text
ActiveRecord::RecordInvalid: Validation failed: Captured must be true or false, Name 'Bulbasaur' is already in use, Attack must be greater than or equal to 0, Defense must be less than or equal to 100, Number must be greater than 0, Poke type 'space' is not a valid Pokemon type
```

Don't forget to test for successful creation as well! Once you have confirmed
that all of your `Pokemon` validations are working correctly, test the
validations for your other models. For instance, to test the `uniqueness`
validation in `PokeMove`, create a `PokeMove` with a known combination of
Pokemon and move, e.g.:

```sh
pm = PokeMove.new(pokemon: Pokemon.first, move: Pokemon.first.moves.first)
```

(Note the alternative syntax of assigning `pokemon` and `move`, which uses
association setters under the hood.) When you try `pm.save!`, you should see
your custom error message that "pokemon cannot have the same move more than
once".

Come up with your own tests for `Item` and `Move` validations, then on to Phase
2!

## Phase 2 - Routes, Controllers, and Jbuilder

In this phase, you will set up your routes enabling your new Rails API backend
to communicate with the React frontend you built in Part I.

### Frontend

For the frontend, download the [full solution to Part I][part-i-solution] and
unzip it into a new directory outside of your Rails pokedex backend directory.
`cd` into that directory and run `npm install && npm start`. Your frontend
should be up and running!

The frontend probably appears somewhat underwhelming at this point: if
everything is running correctly, [`localhost:3000`] just shows a blank page with
a `+` button. That's because it's not getting any information from your backend.
It's time to fix that.

To know what your Rails server needs to do, look at the [README from the Part I
backend][express-backend]. That README identifies the API that your frontend is
expecting: which routes to call, how they change the database, and what they
return. The frontend does not care how that API gets implemented under the hood.
Part I used an Express server to receive requests, interact with the database,
and send up the information; you are using a Rails server. You just need to make
sure that the API remains the same. To put it another way, you don't want to
have to change anything in your frontend for it to work with your new Rails
backend.

### Routes

Start by identifying the nine routes in the README that your backend will
require. As you think about how to define these routes, the first thing to note
is that they all begin with `/api/`. You can achieve that effect by nesting all
of your routes inside an `api` namespace. (Remember that _namespaces_ provide a
way of encapsulating items--of grouping them together--primarily to prevent name
collisions.)

To set up an `api` namespace for your routes, include the following in your
__config/routes.rb__:

```ruby
# config/routes.rb
namespace :api, defaults: { format: :json } do
  # define routes here to include them in the api namespace
end
```

The `defaults: { format: :json }` option tells Rails to look first for a
JSON-based file extension such as `.json.jbuilder` rather than an `html.erb`
file when rendering views for routes in this namespace. For more on namespaces,
see the [Rails Guide to Routing][namespaces].

Now that you have set up your `api` namespace, define the nine routes you need
inside the namespace `do`-block. Remember that `resources` makes it easy to
define RESTful routes; just make sure that you `only` create the RESTful routes
that you actually need. (Creating PATCH routes in addition to PUT routes is
fine.) You will also need to define at least one custom route. Finally, note
that the API requires you to nest a couple of routes.

(For help nesting routes, see the [Rails Guide][nested-routes]. In particular,
check out the section on ["Shallow Nesting"][shallow-nesting]; while you are not
required to use the `:shallow` option here, this would be a good opportunity to
see how it works.)

When you have finished defining the routes, test your work by running `rails
routes`. You should see the following routes matching those specified in the
README:

```sh
           Prefix Verb   URI Pattern                              Controller#Action
api_pokemon_types GET    /api/pokemon/types(.:format)             api/pokemon#types {:format=>:json}
api_pokemon_items GET    /api/pokemon/:pokemon_id/items(.:format) api/items#index {:format=>:json}
                  POST   /api/pokemon/:pokemon_id/items(.:format) api/items#create {:format=>:json}
         api_item PATCH  /api/items/:id(.:format)                 api/items#update {:format=>:json}
                  PUT    /api/items/:id(.:format)                 api/items#update {:format=>:json}
                  DELETE /api/items/:id(.:format)                 api/items#destroy {:format=>:json}
api_pokemon_index GET    /api/pokemon(.:format)                   api/pokemon#index {:format=>:json}
                  POST   /api/pokemon(.:format)                   api/pokemon#create {:format=>:json}
      api_pokemon GET    /api/pokemon/:id(.:format)               api/pokemon#show {:format=>:json}
                  PATCH  /api/pokemon/:id(.:format)               api/pokemon#update {:format=>:json}
                  PUT    /api/pokemon/:id(.:format)               api/pokemon#update {:format=>:json}
```

Once your nine routes (plus 2 PATCH routes) match, move on to the controllers!

### Controllers

Now that the routes are ready, you need to create two controllers, one for
`Pokemon` and one for `Items`. Set up your `Pokemon` controller with

```sh
rails g controller api::pokemon --no-test-framework
```

Make sure to preface your controller's name with the namespace (`api::`)! This
will create an __api__ directory inside __app/controllers/__ and add a
__pokemon_controller.rb__ file to it. As before, the `--no-test-framework` flag
is not essential, but it will keep rails from producing boilerplate test files
that you will not use in this project. Go ahead and create your `Items`
controller too.

#### Pokemon `types`, `index`, and `show`

Open __pokemon_controller.rb__. It should contain a basic skeleton for the
`Api::PokemonController` class. **Note that the namespace (`Api::`) precedes the
basic class name.** Begin by defining a custom `types` action inside your
`Api::PokemonController` class. Look at the README from Part I again to see what
this route should return. Since this action returns static data, there is no
need for any Jbuilder views. Simply grab the data, put it into the expected
format, and return it as JSON directly from the controller using this format:

```rb
render json: <data_to_return>
```

That's it! To test your action, start up your server. Remember from yesterday
that the frontend will proxy requests to the backend on port 5000, so your Rails
server needs to listen on port 5000, not the default port 3000. You could
specify the port on the command line when starting the Rails server with the
`-p` flag, like this:

```sh
rails s -p 5000
```

Always having to specify the port can be a pain, however. Instead, go ahead and
change the default port in __config/puma.rb__. Look for the line `port
ENV.fetch("PORT") { 3000 }` (around line 18). This line specifies that the port
should be set to whatever is stored under the `PORT` environment variable. If
that variable is not set (such as during development), then fall back to the
port specified in the `{}`s, here `3000`. Just change `3000` to `5000`:

```rb
# config/puma.rb
# ...
# Specifies the `port` that Puma will listen on to receive requests; default is 3000.
#
port ENV.fetch("PORT") { 5000 }
# ...
```

Voila! No more need for `-p 5000` on the command line.

After starting your server (`rails s`), go into the directory that houses your
frontend. If it is not already running, start the frontend with `npm start`. In
your browser, navigate to [`localhost:3000`]; it should display the blank page
with the `+` button. Click the `+` to pull up the Create Pokemon form. If your
`types` action is working correctly, the dropdown menu at the bottom of the form
should be filled with the different Pokemon types.

Once your types display correctly in the form, go back to
__pokemon_controller.rb__ and add `index` and `show` actions to your
`Api::PokemonController` class. Remember that all these actions need to do is
find the requested Pokemon--all Pokemon in the case of `index`, the specified
single Pokemon in the case of `show`--save it in an instance variable, and then
render the appropriate view. (How do you know which Pokemon you want to find in
`show`? Hint: think about your routes.) If something goes wrong, render an
appropriate error message and a status code of `:not_found` (i.e., 404) instead.

To complete the `index` and `show` requests, you now need to define views that
will return the requested data in JSON format. It's time for Jbuilder!

### Jbuilder Views

Because __config/routes.rb__ specifies that the `api` namespace defaults to
JSON-formatted responses (see above), Rails will look for an
__<action>.json.jbuilder__ template in a corresponding folder under
__app/views__ when rendering. Of course, since you selected an `api`, `minimal`
build, there is no __views__ folder under __app__. Create one now, then add an
__api__ folder to __app/views__ with additional folders for __pokemon__ and
__items__ inside. In the __pokemon__ folder, add files for each action you want
to render: currently, __index.json.jbuilder__ and __show.json.jbuilder__. Your
file structure should now look like this:

```text
app
├── views
    ├── api
        ├── items
        ├── pokemon
            ├── index.json.jbuilder
            ├── show.json.jbuilder
```

Look at the README for the Part I backend again to see what the `index` route
should return. Once you find the return value, copy it into the top of
__index.json.jbuilder__ and comment it out so you have an easily accessible
record of what you need to return.

The API shows that it should return an array of abbreviated Pokemon objects.
This is exactly what the `array!` command does in Jbuilder: create a top-level
array of objects. You can simply specify which attributes to extract from each
Pokemon in `@pokemon`:

```ruby
json.array! @pokemon, :id, :number, :name, :image_url, :captured
```

Copy that line into __index.json.jbuilder__ and refresh your Pokedex in the
browser to initiate a call to `index`. You should now see the names of the
various Pokemon populate your app, but without any associated images. What
happened to the pictures?

You have several tools to help you see and debug your Jbuilder results:

* the server console (to see which templates are being called and when)
* the logger in the frontend console
* the Redux DevTools
* `debugger`s in your Jbuilder code

Some tools are better at catching certain errors than others, and you will
probably end up using all of these options at some point, so try to keep them
all in mind when you run into trouble. For now, go to the `Redux` tab in your
browser's DevTools and click to view the `State`. Inspect the first `pokemon`
and compare it to the commented-out return value at the top of your Jbuilder
file. Before continuing, find the key that is different. The `pokemon` in your
Redux `State` probably looks similar to this:

```ruby
1:
  id: 1
  number: 1
  name: "Bulbasaur"
  image_url: "1.svg"
  captured: true
```

Did you find the incorrect key? According to the API, the `image_url` key should
be camelCase--`imageUrl`--not the snake_case that your Rails database uses.
Address this issue programmatically by having Jbuilder always convert snake_case
keys to camelCase. To do this, add the following line to the end of
__config/environment.rb__:

```ruby
Jbuilder.key_format camelize: :lower
```

You've changed an environment setting, so you will need to restart your Rails
server for the change to take effect. Now when you refresh your browser and look
at the `State` of your `pokemon` in the DevTools `Redux` tab, you should see the
`image_url` key magically transformed into `imageUrl`. The images should now
appear as well. Great!

As you may remember, the backend in Part I only served up images for captured
pokemon; pokemon who were still in the wild had only a giant `?` for an image.
Jbuilder makes it easy to add this feature. Instead of automatically extracting
`image_url` with the other attributes, first check to see if the Pokemon is
captured. If it is, go ahead and return `image_url`. If not, have the template
return an `image_url` of "/images/unknown.png". (You can use standard Ruby
`if`-clauses inside Jbuilder.)

Next, open __app/views/api/pokemon/show.json.jbuilder__ and copy in a
commented-out version of the API return value you need to create. You don't need
a top-level array this time, so just extract the values from `@pokemon` that you
want included in the Jbuilder object. Don't forget to return the `unknown` image
if the Pokemon is not captured. Remember, too, that your Rails database does
not have columns named `createdAt` and `updatedAt`. How can you access that
information? (Hint: think about `imageUrl`, and be glad that you made a
programmatic change!) Also, note that your previous programmatic change will
**not** help you with `poke_type`; you will need to address that attribute by
explicitly setting the expected key and assigning it the desired value.

Finally, you need to attach an array of `name`s to the key of `moves`. You can
access `@pokemon`'s `moves` through the `has_many` association that you set up
in your model: `@pokemon.moves`. This will return an array of `Moves`. Now you
want to iterate over that collection and create an array of the `name`s nested
under a key of `moves`. Remember that you can use normal Ruby functions in
Jbuilder and that you can always refer to the [Jbuilder docs][jbuilder] for help
if you get stuck.

That's it! When you have finished setting up your Jbuilder `show` file, go back
to your browser and click on one of the Pokemon. It should now pull up all the
basic details for that Pokemon. The items will still be blank, however. Fixing
that issue is your next task.

### Items

You have defined the following routes for `Items`:

```sh
api_pokemon_items GET    /api/pokemon/:pokemon_id/items(.:format)   api/items#index {:format=>:json}
                  POST   /api/pokemon/:pokemon_id/items(.:format)   api/items#create {:format=>:json}
         api_item PATCH  /api/items/:id(.:format)                   api/items#update {:format=>:json}
                  PUT    /api/items/:id(.:format)                   api/items#update {:format=>:json}
                  DELETE /api/items/:id(.:format)                   api/items#destroy {:format=>:json}
```

Now you will write the controller actions and Jbuilder views for these routes.

First, set up your files. Open __app/controllers/api/items_controller.rb__ and
add skeletons to the `Api::ItemsController` class for the four controller
actions you will need. Next, under __views/api__, add a new folder __items__
containing Jbuilder files for `index` and `show`. (`create` and `update` will
both render `show`; `destroy` will not require Jbuilder.) Copy in and comment
out the appropriate API return values for each Jbuilder view.

Begin by implementing the controller action and Jbuilder view for `index`.
Their basic format will be very similar to the `Pokemon` `index` controller
action and view that you just wrote. If you get stuck, you can refer to the
`Pokemon` versions for help, but try to write these actions and views without
looking back. Once you have them implemented, your Pokemon show page should show
the Pokemon's items with their images.

Next, implement `destroy`. All this action needs to do is call `Item.destroy`
with the `id` of the item to delete. As the API notes, you should return a JSON
object with the `id` of the deleted item. No need for Jbuilder here! Just
`render` the `json` return value directly. (If you don't remember how to do
this, look at `Api::PokemonController#types`.) Test your `destroy` action from
the browser to make sure that it works.

For `create` and `update`, set up strong params by defining a private
`item_params` method in `Api::ItemsController`
(__app/controllers/api/items_controller.rb__) that `require`s the item attribute
parameters to be nested under a key of `item` and `permit`s only legitimate
attribute parameters to come through. Then write `update`. It should render the
updated item's `show` view on success. If the update fails, render
`@item.errors.messages` under the `json` key and `:unprocessable_entity` under
`status`.

Now you need to write the Jbuilder `show` view in
__app/views/api/items/show.json.jbuilder__. Note that the return value for
`show` is the same as it is for an individual item in `index`. Keep your code
DRY by writing a partial! Create a new file,
__app/views/api/items/_item.json.jbuilder__, and copy in the code for a single
item from the `index` Jbuilder file. To call this partial, just invoke
`json.partial!` followed by the partial's path within the __views__ folder (omit
the underscore before `_item`). A second argument is a hash specifying the
values of any arguments to be passed (here, `item`). For instance, replace the
copied code in __index.json.jbuilder__ with the following partial call:

```rb
json.partial! 'api/items/item', item: item
```

That one line--appropriately modified--is all you need to include in
__show.json.jbuilder__.

Test your work by editing a Pokemon's item in your browser. You should easily be
able to verify that your `update` action correctly updates an item's `name`,
`happiness`, and `price`. Your frontend, however, currently has no way to update
an item's image file or associated Pokemon. You want to make sure this
functionality works since the API says that it does and other frontend apps
might implement it. To test these additional cases, add the following two lines
to the thunk action creator `updateItem` in your frontend's
__src/store/items.js__ file:

```js
// src/store/item.js
export const updateItem = data => async dispatch => {
  data.imageUrl = "pokemon_berry.svg";
  data.pokemonId = 1;
  // ...
```

In your browser, select a Pokemon other than the first and update an item that
does not have berries for its image. If everything is working correctly, when
you submit the update, the two lines you added to `updateItem` should change the
image to `pokemon_berry` and move the item to the Pokemon with an `id` of 1. As
things currently stand, however, that will not happen: edits you made on the
form will go through, but the image and associated Pokemon will stay the same.

To find out what is wrong, return to __app/controllers/api/items_controller.rb__
and insert a `debugger` inside `item_params`. Try submitting your previous edit
again. When you hit your `debugger` in the server console, type `params`. You
should see something like this (likely with different values for most of the
keys):

```rb
<ActionController::Parameters {"id"=>"17", "happiness"=>15, "name"=>"Super Happy Fun Ball", "price"=>30, "pokemonId"=>1, "imageUrl"=>"pokemon_berry.svg", "format"=>:json, "controller"=>"api/items", "action"=>"update", "item"=>{"id"=>17, "name"=>"Super Happy Fun Ball", "price"=>30, "happiness"=>15}} permitted: false>
```

As you hopefully noticed, the `id`, `name`, `price`, and `happiness` parameters
all appear both 1) at the top level and 2) nested under `items`. Now look back
at `updateItem` in your frontend's __src/store/items.js__ file. Where does your
request nest **anything** under `items`? It doesn't. So where does the `item`
parameter with its hash of nested attributes come from???

The answer, of course, is Rails magic. By default, when Rails receives a request
in **JSON format**, it figures out the likely model from the controller name and
then wraps parameters that correspond to that model's attributes under the model
name (here, `item`). (This is why `item_params` can require `item` even if the
frontend doesn't nest its JSON arguments under that key.) In other words, when
Rails gets the request from your frontend, it recognizes that `id`, `name`,
`price`, and `happiness` are all `Item` attributes and accordingly copies them
under a key of `item`, thereby enabling `item_params` to draw them forth.

Why doesn't Rails grab `imageUrl` and `pokemonId` too? It leaves those
parameters alone because they don't correspond to columns in the `Items` table:
the table's columns are `image_url` and `pokemon_id`. Just as you needed to
convert snake_case to camelCase when sending data back to the frontend, so you
need to convert camelCase to snake_case when receiving requests.

To perform this conversion programmatically, open
__app/controllers/application_controller.rb__. Add a private method to
convert params to snake_case and have it run before `create` and `update`:

```rb
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :snake_case_params, only: [:create, :update]

  private
  def snake_case_params
    params.deep_transform_keys!(&:underscore)
  end
end
```

(Putting this code in `ApplicationController` will make it accessible to both
`Api::ItemsController` and `Api::PokemonController`.)

Type `c` (or `continue`) in your server console so that the server is once again
ready for requests, then submit your item edit request again. This time when you
look at the `params` in the server console, you should see something similar to
this:

```ruby
<ActionController::Parameters {"id"=>"17", "happiness"=>15, "name"=>"Super Happy Fun Ball", "price"=>30, "pokemon_id"=>1, "image_url"=>"pokemon_berry.svg", "format"=>:json, "controller"=>"api/items", "action"=>"update", "item"=>{"id"=>17, "name"=>"Super Happy Fun Ball", "price"=>30, "happiness"=>15}} permitted: false>
```

Hmmmm... that's still not quite what you wanted. Everything is now snake_case
(Yea!), but `image_url` and `pokemon_id` are still not nested under `item`. Can
you guess why?

The answer is that Rails wraps the parameters **before** your snake_case
conversion happens. A robust fix would accordingly involve setting up an
initializer to go through and transform everything to snake_case before the
wrapping occurs, but that's a little outside the scope of this project. Instead,
simply tell Rails to include `imageUrl` and `pokemonId` as parameters to nest by
adding the following line to `Api::ItemsController`:

```rb
# app/controllers/api/items_controller.rb
class Api::ItemsController < ApplicationController
  wrap_parameters include: Item.attribute_names + ['imageUrl', 'pokemonId']
  # ...
end
```

Now when you submit your edit form, `params` should look correct:

```rb
<ActionController::Parameters {"id"=>"17", "happiness"=>15, "name"=>"Super Happy Fun Ball", "price"=>30, "pokemon_id"=>1, "image_url"=>"pokemon_berry.svg", "format"=>:json, "controller"=>"api/items", "action"=>"update", "item"=>{"id"=>17, "name"=>"Super Happy Fun Ball", "price"=>30, "happiness"=>15, "image_url"=>"pokemon_berry.svg", "pokemon_id"=>1}} permitted: false>
```

Note the order in which things happen. When the JSON request comes in, Rails
grabs the parameters to wrap (now including `imageUrl` and `pokemonId`) and
duplicates them under the `item` key. Then, before `update` runs, Rails runs
`snake_case_params`, which converts all the parameters--even those that are
nested!--to snake_case. Finally, `update` runs and calls `item_params`.

Allow the request to finish processing. The edited item should now appear under
the first Pokemon (Bulbasaur) and have an associated berry image. Success!

(Don't forget to remove the two `data` lines that you added to your frontend's
`updateItem`--__src/store/items.js__--for testing purposes.)

Now finish your item controller by writing the `create` action. According to
the API, an image_url will be supplied if `image_url` is not provided.
Accordingly, add this line **after** you have created the new `Item`:

```rb
@item.image_url ||= %w(pokemon_berry.svg pokemon_egg.svg pokemon_potion.svg pokemon_super_potion.svg).sample
```

As with `update`, render the `show` view on successful creation and
`@item.errors.messages` with an appropriate error `status` on failure. Once you
have finished, test your action from the frontend to make sure it works.

### Pokemon `create` and `update`

Finish the project by filling out `create` and `update` for
`Api::PokemonController`. Try to do it without looking at your items controller,
although you can look back if you get stuck. Remember that you will need to add
a `wrap_parameters` line at the top of the class definition to include the one
camelCase parameter. You will also need to add `type` to the list of parameters
to wrap because it is not a `Pokemon` column name.

In fact, `type` requires a little more attention. Since `type` is a reserved
word in Rails, you had to name your `Pokemon` table column `poke_type`, but the
API lists `type` as the appropriate key. The simplest fix for this discrepancy
is to add an alias in the `Pokemon` model that will map any attribute named
`type` onto `poke_type`:

```rb
# app/models/pokemon.rb
class Pokemon < ApplicationRecord
  # ...
  alias_attribute :type, :poke_type
  # ...
end
```

Note that you also need to save/update `moves` separately: `moves` is not a
column in the `pokemons` table, so `Pokemon.new` will not create any `moves`. (A
corollary: you do not need to include `moves` as a permitted attribute in
`pokemon_params`. If you added it, go ahead and remove it.) Here are a few
things to keep in mind as you think about how to save/update the Pokemon's
`moves`:

* You can access the `moves` passed from the frontend through `params`.
* You can use assignment (`=`) with an association and Rails will automatically
  perform the necessary adjustments--both additions and deletions--on the
  intervening join table.
* You might find the [`find_or_create_by`] Active Record Relation method helpful.
* If saving/updating the `moves` fails, the whole create/update should fail
  without persisting anything to the database. To handle this possibility, wrap
  all of your related database operations in a [`transaction`].  

You've already written a Jbuilder `show` template for a Pokemon, so nothing more
is required to render a newly created/updated Pokemon. Now that you know how to
do partials in Jbuilder, however, go ahead and refactor your code so that the
elements common to the `index` and `show` templates reside in a partial. Once
you've finished, test your work in your browser to make sure everything still
renders correctly.

The only thing left to do is render the errors and an `unprocessable_entity`
status if `create` or `update` fails. You have relative freedom on the error
format for `update` because the frontend doesn't do anything with those errors.
The Create Pokemon form, in contrast, is set up to render error messages, so you
need to match the expected format. (Usually this process goes the other way,
with a frontend developer trying to interpret the format produced by the
backend, but the practice is good either way.) For example, if you try to create
a Pokemon with a pre-existing number, the form should show "Number: '1' is
already in use" under the Number field. Or if you send a blank move, it should
report "Moves: can't be blank" under the Moves fields.

Look at the frontend code to figure out the expected format for your errors.
Things to consider as you try to match that format include:

* Should you use `errors.messages` or `errors.full_messages`? (What is the
  difference?)
* Should you use new/save, or create, or the ! versions of those methods?
  (Again, what is the difference?)
* How will you handle error keys that do not correspond exactly to the keys
  expected by the frontend?

As you test, try to trigger validation errors for every field at least once.

This last task can be quite challenging, so don't worry if you find it
difficult. Well-deserved kudos, though, if you are able to get everything
working!

## What you've learned

In today's project, you built a Rails API backend server from scratch,
investigated how Rails handles parameters, and experienced some of the
challenges that arise when trying to make your code interface with a
pre-existing app. This enabled you to refresh your Ruby/Rails skills while also
practicing new syntactic sugar and patterns like Jbuilder that will serve you
well in your full stack project.

[rails-api]: https://guides.rubyonrails.org/api_app.html
[length-validation]: https://guides.rubyonrails.org/active_record_validations.html#length
[numericality]: https://guides.rubyonrails.org/active_record_validations.html#numericality
[has-many-through]: https://guides.rubyonrails.org/association_basics.html#the-has-many-through-association
[custom-error-messages]: https://guides.rubyonrails.org/active_record_validations.html#message
[scope]: https://guides.rubyonrails.org/active_record_validations.html#uniqueness
[part-i-solution]: https://aa-swe-online-mca-solution-zips.s3.us-west-2.amazonaws.com/practice-for-week-15-react-thunk-pokedex-long-practice.zip
[`localhost:3000`]: http://localhost:3000
[express-backend]: https://github.com/appacademy/practice-for-week-15-pokedex-express-backend
[namespaces]: https://guides.rubyonrails.org/routing.html#controller-namespaces-and-routing
[nested-routes]: https://guides.rubyonrails.org/routing.html#nested-resources
[shallow-nesting]: https://guides.rubyonrails.org/routing.html#shallow-nesting
[jbuilder]: https://github.com/rails/jbuilder
[`find_or_create_by`]: https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/Relation.html#method-i-find_or_create_by
[`transaction`]: https://api.rubyonrails.org/v6.1.4/classes/ActiveRecord/Transactions/ClassMethods.html