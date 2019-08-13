---
title: Internationalization in a React Application
date: '2019-08-12'
description:
  "Adding multi-language support to a large React project can be tricky. Here's
  a simple way to achieve it using React's context."
tags: ['javascript', 'react']
---

If you are working on an application that will require multiple-language
support, then you've probably already given some thought to
[internationalization and localization](https://en.wikipedia.org/wiki/Internationalization_and_localization).
Whether you have or not, here is my suggestion:

**_Do not wait until you need additional languages to implement localization in
your application._**

Implementing an effective localization framework early will save you from
mountains of mindless refactoring and copy-pasting later on.

> This article will reference the following repositories and articles. You can
> check them out before reading, but they will also be linked below.
>
> - [React's Context API](https://reactjs.org/docs/context.html) - a data
>   sharing system within React
> - [AirBnB's Polyglot.js library](https://airbnb.io/polyglot.js/) - an
>   internationalization helper library
> - [Localize Toolkit](https://github.com/xneelo/localize-toolkit) - the final
>   product

<!-- First, let's take a look at some requirements for this localization system, and
then figure out how we are going to implement it. -->

## Starting Off Simple

At the most simple, a localization framework takes some sort of key and returns
the phrase for it in the current language. This could look something like this:

```jsx
import React from 'react';

let language = 'en';

const englishPhrases = {hello_world: 'Hello World!'};

const availablePhrases = {en: englishPhrases};

const t = key => {
  const phrases = availablePhrases[language];
  return phrases[key];
};

const LocalizedComponent = () => {
  return <p>{t('hello_world')}</p>;
};
```

This is great, but you will notice that it's not really the "React way". If the
language changes while LocalizedComponent is mounted, it won't update the
`t('hello_world')`. We can solve this issue by using
[React's Context API](https://reactjs.org/docs/context.html).

## The React Way

Ok, so let's take a crack at it using context. First off, we will use the same
simple function we defined before and build off of that.

```jsx
import React from 'react';

const LocalizeContext = React.createContext({
  setLanguage: () => {},
  t: () => {},
});

const englishPhrases = {hello_world: 'Hello World!'};

const availablePhrases = {en: englishPhrases};

const LocalizeProvider = ({children}) => {
  const [language, setLanguage] = React.useState('en');

  const t = key => {
    const phrases = availablePhrases[language];
    return phrases[key];
  };

  return (
    <LocalizeContext.Provider value={{setLanguage, t}}>
      {children}
    </LocalizeContext.Provider>
  );
};

const LocalizedComponent = () => {
  const localize = React.useContext(LocalizeContext);

  return <p>{localize.t('hello_world')}</p>;
};
```

A little more complicated, but not too bad. The provider context exposes the
same `t` function as before, but also exposes a function `setLanguage` in order
to change the language the "React way" (in a way React can detect and trigger
updates for). Let's break it down.

```jsx{3-6}
import React from 'react';

const LocalizeContext = React.createContext({
  setLanguage: () => {},
  t: () => {},
});
```

Here we create the context. We put placeholders in for the two functions just in
case you try to use the context outside of the `LocalizeProvider` we just
constructed. To avoid this, you should make sure to wrap your root App component
in the provider so that it is usable all the way throughout your application.

```jsx{2,10-12}
const LocalizeProvider = ({children}) => {
  const [language, setLanguage] = React.useState('en');

  const t = key => {
    const phrases = availablePhrases[language];
    return phrases[key];
  };

  return (
    <LocalizeContext.Provider value={{setLanguage, t}}>
      {children}
    </LocalizeContext.Provider>
  );
};
```

You can see that the language has been moved into a React state instead of being
a variable. This allows you to set the language in a way React can detect: using
the `setLanguage` function exposed by `useState`.

Also notice how both `setLanguage` and `t` are being set in the value prop of
the provider. Anything that consumes the context has access to both of those
functions, which means that any child can set the language, or translate a key.

## Great, Let's Make It Better

So hooks have unlocked a bunch of ways to simplify how components are written,
while still selectively optimizing how functions within them are memoized. While
this is useful for optimization, it is extremely important, even vital when
using context. When components consume functions or variables from the context
value, and then use them inside of a `useEffect`, it's important that these
values do not update unless they absolutely need to, as every update will
trigger the `useEffect`. Here are some optimizations we can do (check the
comments for information).

```jsx
import React from 'react';

const LocalizeContext = React.createContext({
  setLanguage: () => {},
  t: () => {},
});

const englishPhrases = {hello_world: 'Hello World!'};

const availablePhrases = {en: englishPhrases};

const LocalizeProvider = ({children}) => {
  // 1. This is actually perfect as it is. The `setLanguage` function remains
  //    static, as React.useState is optimized this way. Returning it for use
  //    inside a `useEffect` will not cause it to reload.
  const [language, setLanguage] = React.useState('en');

  // 2. Memoizing the function `t` ensures that it will remain the same to
  //    equality checks unless the language changes. This is perfect, as we want
  //    the function to be called again every time the language changes so that
  //    an updated translation is displayed.
  const t = React.useCallback(
    key => {
      const phrases = availablePhrases[language];
      return phrases[key];
    },
    [language],
  );

  // 3. Memoizing the value returned will ensure that components using
  //    LocalizeContext will only reload when the value changes, which is only
  //    when either `setLanguage` or `t` change, which is only when `language`
  //    changes since `setLanguage` will remain the same and `t` will only ever
  //    change when language does. Therefore, components that consume from the
  //    context will only automatically update every time language changes.
  const value = React.useMemo(() => ({setLanguage, t}), [setLanguage, t]);

  return (
    <LocalizeContext.Provider value={value}>
      {children}
    </LocalizeContext.Provider>
  );
};

const LocalizedComponent = () => {
  const localize = React.useContext(LocalizeContext);

  return <p>{localize.t('hello_world')}</p>;
};
```

These changes ensure that we are optimizing our updates, which is very important
since it's likely that most components in your application will be utilizing the
`LocalizeContext`.

In our first pass using context, all components that consume the context would
reload every time anything in your React tree changed, since the changes would
propagate up to the `children` prop, causing the provider to reload (which wraps
all of your components), which would create a brand new object for `value`,
causing a refresh in all children.

With these optimizations, changes to children do nothing to change the `value`
object, only changes to the language. This means we will avoid unnecessary
reloads, and only update children when the language changes. This is exactly
what we want: dynamic updates to all translated strings every time the language
changes.

## But Wait, There's More

You'd be fine with that if you don't require any complexity beyond simple
translation. However there are a few things lacking in this 33 line example that
you'll find in the
[final product (Localize Toolkit)](https://github.com/xneelo/localize-toolkit):

1. A system for organizing phrases would be beneficial. In this example, they
   all reside at the base level of their object. It would be nice to be able to
   nest the object in order to organize based on page or common uses.
1. It would be nice to be able to use templating strings to insert dynamic
   content into the translated phrase. For example, the key `hello_name` could
   map to the phrase `'Hello %{name}'` and allow you to set name to be
   `'John Doe'`.
1. It would be nice to fetch phrases from an API. This would save space on the
   front-end by storing the large phrase objects on the back-end and fetching
   them as needed.
1. It would be nice to have a caching system for storing fetched phrases in
   order to avoid unnecessary API calls to fetch new languages.

The first two points are solved by the localization framework itself. In the
case of the [Localize Toolkit](https://github.com/xneelo/localize-toolkit), this
meant including [AirBnB's Polyglot.js library](https://airbnb.io/polyglot.js/).
While all of these things are achievable yourself, sometimes it's better to use
someone's pre-tested, proved solution. In the case of Polyglot.js, this means
you get nesting and organization out of the box, as well as string interpolation
and pluralization. Will I likely implement my own solution down the road in the
Localize Toolkit? Probably, but it is a lot of work for something that is
already built and works perfectly.

The last two issues can easily be solved by adding functions within the body of
the `LocalizeProvider`. I encourage you to browse the code and documentation for
the [Localize Toolkit](https://github.com/xneelo/localize-toolkit) to see how
they were implemented.

## That's It

Ok, that pretty much does it. It's not a very complex tool, but it provides you
with an efficient and versatile system for implementing multiple languages in
your application. Obviously the actual toolkit is a fair bit more complex and
exposes more functions and variables, but in essence the use case is the same.

Good luck with your localization.