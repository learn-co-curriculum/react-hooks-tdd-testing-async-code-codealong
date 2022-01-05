# Testing Asynchronous Code Codealong

## Learning Goals

- Understand the challenges of testing asynchronous code
- Use `findBy` queries to test find elements asynchronously

## Introduction

Working with JavaScript often means handling _asynchronous_ events in our code
so that we can use some of the browser's most powerful features, like scheduling
events using `setInterval` and `setTimeout`, or making network requests using
`fetch`.

Any time our code runs asynchronously, it presents some challenges for testing:
instead of running our tests line-by-line, one step at a time, and making
assertions that _as soon as a button is clicked_ the application's state might
change, we need to instruct our tests to _wait for some unknown amount of time_
before asserting that the application works as expected.

Thankfully, React Testing Library has a few handy features that simplify the
process for us. To demonstrate, we'll work on writing some tests for an example
application together.

Fork and clone this lesson, and run `npm install` to get started.

## The Demo App

The demo application we're building is a translation service that uses the free
[LibreTranslate][libre translate] API to translate text from one language to
another.

We've built out the complete functionality, but we still need to finish writing
tests.

You can play around with the demo app by running `npm start` and interacting
with the site in the browser. Typing text and pressing the "Translate" button
will:

- Make a request to the API,
- Update the component's state when a response comes back with the translation,
  and
- Re-render the component with the translated text

Take a look at the code in the `src/App.js` file to see how this is put
together. Essentially, when the form submits, we take the data from the form
itself and make an API request, then update state with the response data to
re-render the component and show the translation.

You can also run the tests with `npm test`. Our last test is **failing**, even
though we can see in the browser that the component is behaving correctly. In
this case, our test is failing because of an issue with the test itself! Let's
see why this is, and how to fix it.

> **Note**: The API we're using for this lesson is public and free to use, but
> it's not maintained by the authors of this course. That means that if the API
> is slow, or doesn't work properly, you may run into issues with this lesson.
> If that's the case, just read this lesson to help understand the concepts. In
> the next lesson, we'll find a way around this issue!

## Challenges of Testing Async Code

To understand why the test is failing, let's take a look at the test itself:

```jsx
test("translates the text when the form is submitted", () => {
  render(<App />);

  // Find the form input fields
  const languageFrom = screen.getByLabelText(/^from$/i);
  const languageTo = screen.getByLabelText(/^to$/i);
  const textFrom = screen.getByLabelText(/text to translate/i);
  const submitButton = screen.getByRole("button", { name: /translate/i });

  // Fill out the form and submit
  userEvent.selectOptions(languageFrom, "en");
  userEvent.selectOptions(languageTo, "es");
  userEvent.type(textFrom, "Hello.");
  userEvent.click(submitButton);

  // Assert that the translated text appears on the page
  const textTo = screen.getByDisplayValue("Hola.");
  expect(textTo).toBeInTheDocument();
});
```

When this test runs, we act on it from the perspective of a user, just like we
would in the browser: we use the form elements to select the language, type in
our text to be translated, and submit the form. When the form submits, we know
that a network request will be made to the API and our component will re-render
to show the new data, so we expect to see the translated text ("Hola.") appear
on the page. However, that expectation is failing:

```txt
TestingLibraryElementError: Unable to find an element with the display value: Hola.
```

The reason for this is because our test is running _synchronously_. As soon as
the button is clicked to submit the form, the next line of the test runs, and we
expect the new text to appear on the page _immediately_. However, we know that
network requests _aren't synchronous_: any time we need to make a network
request, it will take some unknown amount of time to complete. Therefore, we
need to treat this as an _asynchronous action_ and handle this part of our test
a bit differently.

## Use findBy Queries For Asynchronous Actions

To fix the problem with our tests, we need to instruct our tests to _wait_ some
amount of time until the component re-renders, and the element we expect to be
on the page is present.

Thankfully, React Testing Library makes this quite simple to do! All we need to
do is:

- Change the `getBy` query to a `findBy` query
- Use Promise-handling syntax (`.then` or `async/await`)

From the documentation, the `findBy` query does the following:

> Returns a Promise which resolves when an element is found which matches the
> given query. The Promise is rejected if no element is found or if more than
> one element is found after a default timeout of 1000ms.

What that means is that we can rewrite out test like this if we want to instruct
the tests to wait for up to 1000ms (1 second) for the element to appear on the
page.

Here's how we can use it:

```jsx
test("translates the text when the form is submitted", () => {
  render(<App />);

  // Find the form input fields
  const languageFrom = screen.getByLabelText(/^from$/i);
  const languageTo = screen.getByLabelText(/^to$/i);
  const textFrom = screen.getByLabelText(/text to translate/i);
  const submitButton = screen.getByRole("button", { name: /translate/i });

  // Fill out the form and submit
  userEvent.selectOptions(languageFrom, "en");
  userEvent.selectOptions(languageTo, "es");
  userEvent.type(textFrom, "Hello.");
  userEvent.click(submitButton);

  // Assert that the translated text appears on the page
  return screen
    .findByDisplayValue("Hola.")
    .then((textTo) => expect(textTo).toBeInTheDocument());
});
```

The syntax above is a bit harder to read than our original test, and it'll get
even messier if we need to handle multiple asynchronous actions. We can write
this more cleanly using the [`async/await`][async await] syntax:

```jsx
// IMPORTANT: Add the async keyword as part of the callback function definition
test("translates the text when the form is submitted", async () => {
  render(<App />);

  // Find the form input fields
  const languageFrom = screen.getByLabelText(/^from$/i);
  const languageTo = screen.getByLabelText(/^to$/i);
  const textFrom = screen.getByLabelText(/text to translate/i);
  const submitButton = screen.getByRole("button", { name: /translate/i });

  // Fill out the form and submit
  userEvent.selectOptions(languageFrom, "en");
  userEvent.selectOptions(languageTo, "es");
  userEvent.type(textFrom, "Hello.");
  userEvent.click(submitButton);

  // Assert that the translated text appears on the page
  // Use the await keyword instead of .then
  const textTo = await screen.findByDisplayValue("Hola.");
  expect(textTo).toBeInTheDocument();
});
```

This syntax looks a lot more like our original example, but crucially, the value
of `textTo` won't be assigned unto the component re-renders and an element
containing the text `"Hola"` is present.

Now as long as we get a response from the API within 1000ms, and the text is
correctly displayed by our components, our tests will all pass!

A nice benefit to this approach of using the `findBy` query is we're not testing
implementation details: our tests don't care that an API request is being made,
or that our component state is changing. All our tests care about (and the same
is true of our users!) is that the translated text shows up on the page,
eventually.

React Testing Library has a number of other helpful methods for dealing with
async code in other scenarios. In our example, the `findBy` query is appropriate
since we need to wait for a particular element to appear on the page. You may
want to wait for other things to happen in your tests, like asserting that an
element was _removed_ from the page, or spying on an asynchronous function.
Check out the [docs on Async Methods][async methods] for more info.

## Conclusion

Working with asynchronous code is quite common for JavaScript developers because
it enables many powerful features of the language, like making network requests.
However, it adds an additional layer of complexity to our applications and needs
to be handled with care. With React Testing Library, the `findBy` query methods,
combined with the `async/await` syntax, make it relatively easy to test that an
element has been added to the page asynchronously.

While we were able to get our tests passing again, we've got one big problem
with our tests: they rely on an external API in order to work correctly. In the
next lesson, we'll discuss why this is a problem and show how to fix it.

## Resources

- [Testing Library - Async Methods][async methods]
- [`async/await` syntax][async await]

[libre translate]: https://libretranslate.com/
[async await]: https://javascript.info/async-await
[async methods]: https://testing-library.com/docs/dom-testing-library/api-async
