# Hubot Markov Model

[![npm](https://img.shields.io/npm/v/hubot-markov.svg?style=plastic)](https://www.npmjs.com/package/hubot-markov) [![Travis branch](https://img.shields.io/travis/smashwilson/hubot-markov/master.svg?style=plastic)](https://travis-ci.org/smashwilson/hubot-markov)

Generates a markov model based on everything that your Hubot sees in your chat.

## Installing

1. Add `hubot-markov` to your `package.json` with `npm install --save hubot-markov`:

```json
  "dependencies": {
    "hubot-markov": "~2.0.0"
  },
```

2. Require the module in `external-scripts.json`:

```json
["hubot-markov"]
```

3. Run `npm update` and restart your Hubot.

Consult the [upgrading guide](./docs/upgrade.md) for instructions on migrating from older major versions.

## Commands

By default, saying anything at all in chat trains the model. The robot is always watching!

`Hubot: markov` will randomly generate text based on the current contents of its model.

`Hubot: markov your mother is a` will generate a random phrase seeded with the phrase you give it. This command might output "your mother is a classy lady", for example. Remember: Hubot is an innocent soul, and what he says only acts as a mirror for everything in your hearts.

`Hubot: remarkov` and `Hubot: mmarkov` are similar, but traverse node transitions in different directions: `remarkov` chains backwards from a given ending state, and `mmarkov` chains both forward and backward.

## Configuration

The Hubot markov model can optionally be configured by setting environment variables:

* `HUBOT_MARKOV_DEFAULT_MODEL` _(default: true)_ controls the inclusion of the default, forward-chaining model that learns from all text messages. Set this to `false` to omit the default model and disable the `markov` and `mmarkov` commands.

* `HUBOT_MARKOV_REVERSE_MODEL` _(default: true)_ controls the inclusion of the reverse model. Setting this to `false` saves some space in your database, but doesn't let you use `remarkov` or `mmarkov`.

* `HUBOT_MARKOV_PLY` _(default: 1)_ controls the *order* of the default models that are built; effectively, how many previous states (words) are considered to choose the next state. You can bump this up if you'd like, but the default of 1 is both economical with storage and maximally hilarious.

* `HUBOT_MARKOV_LEARN_MIN` _(default: 1)_ controls the minimum length of a phrase that will be used to train the default models. Set this higher to avoid training your model with a bunch of immediate terminal transitions like "lol".

* `HUBOT_MARKOV_GENERATE_MAX` _(default: 50)_ controls the maximum size of a markov chain that will be generated by the `markov`, `remarkov`, and `mmarkov` commands.

* `HUBOT_MARKOV_STORAGE` _(default: memory)_ controls the backing storage used to persist the default models. Choices include:
  * `memory`, the default, which stores transitions entirely in-process (lost on restart);
  * `redis`, which stores data in a Redis cache; or
  * `postgres`, which stores data in a PostgreSQL database.

* `HUBOT_MARKOV_STORAGE_URL` supplies additional configuration required by the `redis` and `postgres` storage backends. The formats are `redis://${USER}:${PASSWORD}@${HOSTNAME}:${PORT}/${DBNUM}` and `postgres://${USER}:${PASSWORD}@${HOSTNAME}:${PORT}/${DATABASE}` with defaults omitted.

* `HUBOT_MARKOV_RESPOND_CHANCE` controls the chance that Hubot will respond un-prompted to a message it sees by using the last word in the message as the seed. Set this to a value between 0 and 1.0 to enable the feature. Leaving this variable unset or setting it to 0 will disable the feature.

* `HUBOT_MARKOV_INCLUDE_URLS` _(default: false)_ will default to ignoring messages that include URLs from the default models.

* `HUBOT_MARKOV_IGNORELIST` _(default: empty)_ is interpreted as a comma-separated list of usernames to ignore for purposes of markov indexing. You can use this to prevent the output of other bots or integrations from clogging up your model.

To re-use a PostgreSQL connection with other parts of your Hubot, define a robot method called `getDatabase` that returns the connection object. This package uses [pg-promise](https://www.npmjs.com/package/pg-promise).

## Custom models

Store and generate text from arbitrary sources and in more complex commands by using the programmatic API available at `robot.markov`. Call `robot.markov.createModel` during script initialization to configure a model, then use `robot.markov.modelNamed` to access the model instance in commands that train it or generate from it.

### Example: Manual Model

```coffee
module.exports = (robot) ->
  MODELNAME = 'manual'

  # Create or connect to a model with all default options
  robot.markov.createModel MODELNAME

  robot.respond /modeladd\s+(.+)/, (msg) ->
    robot.markov.modelNamed MODELNAME, (model) ->
      model.learn msg.match[1], ->
        msg.reply 'Input accepted.'

  robot.respond /modelgen(?:\s+(.+))/, (msg) ->
    robot.markov.modelNamed MODELNAME, (model) ->
      model.generate msg.match[1] or '', 50, (output) ->
        msg.reply output
```

### Example: Letter-Based Model

```coffee
module.exports = (robot) ->
  MODELNAME = 'letters'

  # Create or connect to a model with a custom pre- and post-processor
  robot.markov.createModel MODELNAME, {}, (model) ->
    model.processWith
      pre: (input) -> input.split('')
      post: (output) -> output.join('')

  robot.catchAll (msg) ->
    # Filter out "lol"
    return if /^\s*l(o+)l\s*/.test msg.text

    robot.markov.modelNamed MODELNAME, (model) ->
      model.learn msg.text

  robot.respond /lettergen(?:\s+(.+))/, (msg) ->
    robot.markov.modelNamed MODELNAME, (model) ->
      model.generate msg.match[1] or '', 100, (output) ->
        msg.reply output
```

The full API is available in [the docs/ directory.](./docs/api.md)