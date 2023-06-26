# Pokedex, Part 1: Backend

In today's project, you will create a lean Rails backend API for Pokedex,
basically, a database of Pokemon. Along the way, you will
learn some new syntactic sugar, investigate how Rails handles parameters, and
practice building Jbuilder templates.

(In Pokedex, Part 2--which you will do on another day--you will build a
corresponding frontend to access your API.)

## Phase 1 - Create a new Rails project and set up the database

To begin, create a new Rails project:

```shell
rails new pokedex-rails-backend -G -T --database=postgresql --api --minimal
```

Recall that the `-G` flag keeps Rails from installing a new git repo in the
project; if you want the new git repo installed--make sure it won't create
nested repos!--you can omit the `-G`. (You can grab appropriate
__.gitattributes__ and __.gitignore__ files from the repo linked to the
`Download Project` button at the bottom of this page.)

The `-T` flag tells Rails to skip setting up its internal `Test::Unit` files.

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
  * `faker`
  * Change the `debug` gem to `byebug`

* Under `group :development`
  * `annotate`
  * `pry-rails`
  * `better_errors`
  * `binding_of_caller`

Run `bundle install`.

### Setting up the database

To set up your database, make sure Postgres is running, then run `rails
db:create`. Now you need to fill the database with the appropriate tables. You
will need 4: `pokemons`, `items`, `moves`, and `poke_moves`. Run the following
command to generate a migration for the `pokemons` table and model:

```shell
rails g model Pokemon number:integer:uniq name:string:uniq attack:integer defense:integer poke_type:string image_url:string captured:boolean
```

Take a minute to look at the migration file that this command creates in
__db/migrate__ and refresh your memory of Rails. Notice that appending `:uniq`
to `number:integer` and `name:string` effectively added indexes for `number` and
`name` with uniqueness constraints. Note, too, that Rails has automatically
added `t.timestamps`. Hopefully this feels like familiar territory.

Go ahead and add `null: false` constraints to the other fields (`number`,
`name`, `attack`, `defense`, `poke_type`, `image_url`, and `captured`). Set
`captured` to default to `false` as well. Note that while setting a default
value on `captured` will assure that the field is not `null` in most cases, it
is still useful to have a `null: false` constraint to protect against instances
where `captured` is explicitly set to `null`. This could happen, for instance,
if an attempt to set `captured` dynamically fails.

Next create a migration for your `items` table and model. It should have the
following column names and types:

* pokemon (type: references)
* name (type: string)
* price (type: integer)
* happiness (type: integer)
* image_url (type: string)

Open the newly created migration file. Add `null: false` constraints to the
`name`, `price`, `happiness`, and `image_url` columns. Remember that the
`references` type is used to create foreign keys; when you run the migration,
the `references` type will create a `pokemon_id` column (N.B.: **NOT**
`pokemon`) and add an index for it.

Recall, too, that the `foreign_key` constraint will prevent your app from
deleting records needed by other tables. To ensure that deleting a Pokemon will
automatically delete any associated items first, you will want to add
`dependent: :destroy` to the corresponding `has_many` relationship in the model.
(You will be instructed to do this below.)

Finish creating your migrations by generating migrations for the `moves` and
`poke_moves` models. `moves` needs only a unique string `name`. Don't forget to
add a `null: false` constraint in the migration file!

`poke_moves` is a join table connecting a `move` with a `pokemon`. Create this
migration on the command line using the `references` type, then add a constraint
to ensure that each move associated with a given Pokemon is unique. To do this,
go into the migration file and add a composite index on `pokemon_id` and
`move_id` with a uniqueness constraint.

> **Note**: When creating the index, order matters. Although Postgres will now
> switch the order of an inquiry to match the order of a composite index, it is
> still worth considering whether it would be more efficient to search for
> moves associated with a given Pokemon or the Pokemon associated with a given
> move.

Since the creation of the composite index will effectively create an index for
the first column, you should also add `index: false` to the `t.references` line
for that foreign key column. This will keep your app from creating two distinct
indexes for that foreign key.

Your `create_table` for `poke_moves` should now look something like this:

```rb
create_table :poke_moves do |t|
  t.references :pokemon, null: false, foreign_key: true, index: false
  t.references :move, null: false, foreign_key: true
  t.index [:pokemon_id, :move_id], unique: true

  t.timestamps
end
```

(**N.B.:** `poke_moves` requires that both the `pokemons` and the `moves` tables
already exist in order to create the foreign keys. This migration must
accordingly run **after** the migrations for those two tables or it will fail.)

Once you have finished creating your four migrations, run `rails
db:migrate`. The resulting __db/schema.rb__ file should look like this:

```rb
ActiveRecord::Schema[7.0].define(version: 2022_08_11_204557) do
  # These are extensions that must be enabled in order to support this database
  enable_extension "plpgsql"

  create_table "items", force: :cascade do |t|
    t.bigint "pokemon_id", null: false
    t.string "name", null: false
    t.integer "price", null: false
    t.integer "happiness", null: false
    t.string "image_url", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["pokemon_id"], name: "index_items_on_pokemon_id"
  end

  create_table "moves", force: :cascade do |t|
    t.string "name", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["name"], name: "index_moves_on_name", unique: true
  end

  create_table "poke_moves", force: :cascade do |t|
    t.bigint "pokemon_id", null: false
    t.bigint "move_id", null: false
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
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
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.index ["name"], name: "index_pokemons_on_name", unique: true
    t.index ["number"], name: "index_pokemons_on_number", unique: true
  end

  add_foreign_key "items", "pokemons"
  add_foreign_key "poke_moves", "moves"
  add_foreign_key "poke_moves", "pokemons"
end
```

If anything went awry, create a migration to fix it.

### Validations and associations

Look at the files in __app/models__; there should be one for each of the 4
models. Run `bundle exec annotate --models` to import the relevant schema into
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
error message by using `%{value}`, `%{attribute}`, and `%{model}`, respectively.
Here, you can use `%{value}` to reference the particular non-unique `name`. (For
more on customizing error messages, see the [Guide][custom-error-messages].)

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
supplied the `belongs_to` associations for each of the foreign keys. For each
`belongs_to`, now add a corresponding `has_many` in the appropriate model.
Remember to add the `dependent: :destroy` option where warranted. Add an
additional `has_many :moves` in the `Pokemon` model that goes through
`poke_moves`. Add the corresponding `has_many :pokemon` in `Move`, too. (For
`has_many :through` associations, see [here][has-many-through].)

### Seed and test

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
`id`s 1, 2, and 3 (among others!).

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

[rails-api]: https://guides.rubyonrails.org/api_app.html
[length-validation]: https://guides.rubyonrails.org/active_record_validations.html#length
[numericality]: https://guides.rubyonrails.org/active_record_validations.html#numericality
[has-many-through]: https://guides.rubyonrails.org/association_basics.html#the-has-many-through-association
[custom-error-messages]: https://guides.rubyonrails.org/active_record_validations.html#message
[scope]: https://guides.rubyonrails.org/active_record_validations.html#uniqueness
