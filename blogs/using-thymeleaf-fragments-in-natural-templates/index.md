# Using Thymeleaf fragments in natural templates

24/11/2015

At Black Pepper we have enjoyed using [Thymeleaf](http://www.thymeleaf.org/) as the templating engine on several of our projects. Its non-invasive syntax and [tight integration with the Spring Framework](http://www.thymeleaf.org/doc/tutorials/2.1/thymeleafspring.html) offers a far more pleasant experience than the bygone days of JSP. The unique feature of Thymeleaf, though, that makes it so attractive for us is its concept of 'natural templates'.

## Natural templates

The idea behind natural templates is that the same file used for templating at runtime can also be displayed correctly in a browser as a static HTML file for prototyping. This means that our UX designers can simply open the same templates that our developers use to work on them without having to run the underlying application. This vastly simplifies the UX workflow and removes the need for designers' environments to be kept development ready.

As well as this works in practise it does start to fall down when using [template fragments](http://www.thymeleaf.org/doc/tutorials/2.1/usingthymeleaf.html#template-layout) to reduce duplication, since there is currently no standard approach to client-side includes in HTML. Third-party tools such as [Thymol](http://www.thymoljs.org/) do help to address this problem by processing Thymeleaf attributes, including fragments, in the browser using JavaScript. Unfortunately this also means that all other attributes are processed too, which somewhat negates its usefulness for prototyping.

## thymeleaf-fragment.js

To this end we decided to write a simple script to only process Thymeleaf fragment attributes in the browser. This has proven to be useful so we have open-sourced it as [thymeleaf-fragment.js](https://github.com/BlackPepperSoftware/thymeleaf-fragment.js). Getting started is straightforward enough: simply include it in your template and use the standard Thymeleaf syntax to include or replace fragments:

```html
<script src="https://code.jquery.com/jquery-2.1.4.min.js" th:if="false"></script>
<script src="http://blackpeppersoftware.github.io/thymeleaf-fragment.js/thymeleaf-fragment.js"
    defer="defer" th:if="false"></script>
```

```html
<html>
    <body>
        <div th:include="fragments::helloworld"></div>
    </body>
</html>
```

For further information, including the subtleties of including fragments locally, head over to the [project's GitHub page](https://github.com/BlackPepperSoftware/thymeleaf-fragment.js) and let us know how you get on.
