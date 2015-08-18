# Natural Language Processing with Components

This forms the basis of Lacona, a [language processing framework](lacona/lacona) that runs behind a [Mac App of the same name](https://www.kickstarter.com/projects/2102999333/lacona-natural-language-commands-for-your-mac).

React is a Javascript framework for declaratively building webpages. The beauty is that the complexities of state management are abstracted away, because components are defined declaratively. Components are composed of other components, which pass props down through to the chain.

## Defining Stuff: Grammar

[Lacona](lacona/lacona) takes this philosophy and applies it to describing natural language. The idea is that in language, phrases are built out of combinations of smaller phrases. `props` are passed *down* the chain to describe how language components work. Once parsing is done, `results` are passed *up* the chain, creating plain Javascript objects to describe information conveyed by the natural language.

### `props`

This syntax, like React, can be (but needn't be) specified using the JSX Javscript extension and compiled using [Babel](http://babeljs.io/). Let's have an example. Let's say I want Lacona to understand commands telling it to tweet things. This is how you would do it:

```js
function describe () {
  return (
    <choice>
      <sequence>
        <literal text='tweet ' category='action' />
        <argument text='message' id='message'>
          <StringPhrase maxlen={140} />
        </argument>
      </sequence>
      <sequence>
        <literal text='tweet ' category='action' />
        <argument text='message' id='message'>
          <StringPhrase maxlen={140} />
        </argument>
        <literal text=' to Twitter' category='action' />
      </sequence>
      <sequence>
        <literal text='post to Twitter ' category='action' />
        <argument text='message' id='message'>
          <StringPhrase maxlen={140} />
        </argument>
      </sequence>
    </choice>
  )
}
```

Notice a few things:

* There is a `describe` function, which is similar to React's `render` method.
* `describe` returns a single component. In this case, a `<choice>`. `<choice>` means "any of these options are valid." In this case, they are all equivalent statements, just said in a different way.
* The `<choice>` has three children, all of which are `<sequence>`s. `<sequence>` means "these elements must occur in order."
* The `<sequences>`'s each have a number of children: either 1 or 2 `<literal>`s, and an `<argument>` that contains a `<StringPhrase>`.
* `<literal>` accepts a specific input, as specified by its `text` prop. `category='action'` specifies that this component is the verb of the sentence, which is used for syntax highlighting.
* `<argument>` represents a named argument that composes a sentence. It has a `text` prop that specifies what it is, and a single child.
* `<StringPhrase>` represents any amount of arbitrary text, optionally confined by certain limitations. In this case, it has a `maxlen` property in order to fit Twitter's limitations.

This structure will understand phrases like "tweet hey guys how's it going?" and "post what's going on folks? to Twitter", but not phrases like "tweet" or "tweet this is way too long this is way too long this is way too long this is way too long this is way too long this is way too long this is way too long".

Of course, understanding language is worthless unless we can convert the language into a machine-readable format. This works through `results`.

### `results`

`results` are passed backwards up through the component chain. Each component has a `results` when parsed. `<literal>`'s `results` is always whatever is in its `value` prop. In this case, `undefined`. The `<StringPhrase>` component's `results` is always whatever text is input by the user.

`<argument>`'s `results` are simply the `results` of its child. Likewise, `<choice>`'s `results` are the results of the child that was selected.

A `<sequence>` has multiple children, so its children's results are composed into a single object using their `id` props. The `id` of the child becomes the key, and its `results` become the value.

So if the user enters "tweet hey guys how's it going?", the final `results` object will be:

```json
{"message": "hey guys how's it going?"}
```

## Doing Stuff: Commands and Extensions

Lacona supports two different kinds of addons - **commands**, which give Lacona new functionality, and **extensions**, which build upon existing functionality. Both commands and extensions are defined using Javascript code.

### Commands

I'll explain by way of example. The developer could make a new command that allows users to tweet. You define two things: the **grammar** (described above) and the **logic** (the code that physically sends the tweet to Twitter). These are defined in an ES6 class definition (though the can easily be described using ES5 syntax). Note that we have abstracted the <

```js
/** @jsx createElement */
import {createElement, Phrase} from 'lacona-phrase'
import StringPhrase from 'lacona-phrase-string'

class TweetContent extends Phrase {
  describe () {
    return (
      <argument text='message'>
        <StringPhrase maxlen={140} />
      </argument>
    )
  }
}

class TweetCommand extends Phrase {
  describe () {
    return (
      <choice>
        <sequence>
          <literal text='tweet ' />
          <TweetContent id='message' />
        </sequence>
        <sequence>
          <literal text='post ' />
          <TweetContent id='message' />
          <literal text='to twitter' />
        </sequence>
        <sequence>
          <literal text='post to Twitter ' />
          <TweetContent id='message' />
        </sequence>
      </choice>
    )
  }

  execute (results) {
    send_to_twitter(results.message)
  }
}
```


That's it - you just specify the `describe` function, and Lacona handles the rest. The `results` parsed are passed to the `execute` function. The developer doesn't need to think about the parsing logic at all.

Lacona users who install your command can now make some insightful tweets.

![Command Structure](https://raw.github.com/lacona/kickstarter-nerd-details/master/images/TweetSentence@2x.png)

A few things you should notice:

- You do not need to do anything fancy to allow the user to input a string that is less than 140 characters - that is all built into Lacona with the `<StringPhrase>` component.
- You are providing many different ways for the user to input the same thing. This flexibility allows anyone to use Lacona in the way that is most natural for them.
- Someone can translate your command, and as long as you they put in a `<StringPhrase>` with `id = message`, it will work with the same code. The `execute` method does not need to change.

You can think of commands as a list of sentences that Lacona will be able to understand. They each have a single verb, and they do a single thing.

## Extensions

Extensions build on top of commands. If commands define new sentences, then Extensions provide new phrases with which you can compose those sentences.

Let's have another example. It may be useful to post the same thing to different social networks. Let's say we want the user to be able enter `tweet my last Facebook status` - a useful thing for those who use multiple social networks.

![Command Structure](https://raw.github.com/lacona/kickstarter-nerd-details/master/images/TweetSentence 1@2x.png)

We already have a Tweet command, but right now, entering `tweet my last Facebook status` would not have the desired results. It would interpret `my last Facebook status` as a string, and post it literally.

![Command Structure](https://raw.github.com/lacona/kickstarter-nerd-details/master/images/TweetSentence@2x.png)

We don't want to rewrite our Tweet Command - we should keep it as general as possible. After all, we wouldn't want to modify the command to fit into every other social network. Instead, we can just extend the `<StringPhrase>` component.

```js
/** @jsx createElement */
import {createElement, Phrase} from 'lacona-phrase'
import StringPhrase from 'lacona-phrase-string'

class FacebookExtension extends Phrase {
  getValue () {
    return get_last_facebook_status()
  }

  describe () {
    return <literal text='my last Facebook status' category='symbol' />
  }
}

FacebookExtension.extends = [StringPhrase]
```

Now, any time a `<StringPhrase>` is used in any Lacona command, `my last Facebook status` will also be an option. Of course, it is possible that the user may want to tweet that exact message, so both options will be presented side-by-side.

![Command Structure](https://raw.github.com/lacona/kickstarter-nerd-details/master/images/TweetSentence 2@2x.png)

If the user selects the italicized, purple option, it will have the desired effect.

![Command Structure](https://raw.github.com/lacona/kickstarter-nerd-details/master/images/TwitterShot 1@2x.png)

Many of the built-in commands use the `<StringPhrase>` component, so the user will now be able to use their last Facebook status in other contexts as well. This may or may not be useful, but the option is always available.

![Command Structure](https://raw.github.com/lacona/kickstarter-nerd-details/master/images/SearchSentence@2x.png)

## Internationalization

Of course, the problem with Natural Language interfaces is that they're fairly difficult to translate. It's not enough to simply translate the labels in a form and call it a day. Here are just a few common design issues.

- Languages have different word order, including the position of verbs
- Some languages have accented/modified characters, which can be essential but are a pain to type
- Some languages have dramatic differences between dialects
- Some languages have multiple sets of characters with the same meaning
- Some languages separate words with spaces, others do not
- Some languages are written left-to-right, some right-to-left
- In some languages, it is common to use foreign words or characters to express certain concepts even if words to represent them do exist in the language
- In some languages, the user enters text in one character set, and replaced with different characters over time
- Some languages have words that simply do not have an equivalent in other languages
- Some languages use distinguish between different verbs depending upon the verb's object, while other languages do not
- Clearly, there are a lot of issues. Because of this, Lacona's language processing is as general as possible. It does not rely on the verb coming at the beginning of a sentence, it does not rely on words being separated by spaces, and it does not rely on any particular character set. It takes any input, interprets it according to its Commands, forms an data structure based upon the input, and sends the data to the command to execute.

This means that the *language* of Commands can be translated, while keeping the *code* the same. Developers do not need to know any other languages, or anything about language processing at all. And translators don't need to know anything about code. Lacona handles all of that.

## Under the Hood

Lacona does not know English. That is to say, it cannot break a sentence into grammatical components, and it cannot infer any meaning from an arbitrary sentence. It understands only the sentences that have been described for it in Commands, and nothing more.

This means that everything that Lacona can do must be translated manually into every new language that it is to support. Lacona will only launch with support for US and UK English. However, all of the Lacona commands will be open-source, so that multilingual users from around the world can add new translations.

To get a bit more technical, you can think of Lacona Commands as closer Formal Grammars, or very glorified Regular Expressions. It takes arbitrary strings as input, and interprets them according to its rules. I am not an expert in computational linguistics, but I can provide a bit more information. the biggest differences are:

- Lacona Commands are not necessarily Regular or Context-Free. Commands are composed of components, which can be thought of as similar to rules in a formal grammar. However complex components can be much more complex - they can even make use of arbitrary code to handle possible inputs.
- Lacona Commands are designed to not only interpret an string after it has been input, but also while it is being input. For example, the string "617-" is very clearly not a valid Phone Number, but it is possible that it is the beginning of a Phone Number. Lacona needs to identify that.
- Lacona command components can be evaluated dynamically, so grammars can change state based on external factors, such as the contents of the clipboard, results of scripts or network requests, or the current date.
- Altogether, this means that Lacona has a language processing system that is general enough to support all of the quirks of language, but simple enough be accessible. Simple commands are easy, and very complex commands are possible.

## Further information

Of course, this is just an introduction. Much more information will be available with the release of the API. If you have further questions, please ask on [Kickstarter](https://www.kickstarter.com/projects/2102999333/lacona-natural-language-commands-for-your-mac) or [Twitter](https://twitter.com/lacona).
