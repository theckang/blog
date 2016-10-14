---
layout: post
title: "Postman Environments"
---

[Postman](https://www.getpostman.com/) is a great project to work with APIs.  If you aren't already using it, then you should take a look right away.

When you use Postman, at some point you will probably run into the problem of switching between development and production APIs.  For example, I have a project called `Python Jobserver` with four endpoints.

![](/assets/posts/postman_1.png)

In this example, the API points to my development server on localhost.

What if I want to test the API on my production server?  I could manually switch the URL to my production server like this.

![](/assets/posts/postman_2.png)

But you don't want to manually switch back and forth.  You could copy the entire project and rewrite all the endpoints, but that's really cumbersome too.

There is a much better solution to this in Postman, called [Postman Environments](https://www.getpostman.com/docs/environments).  With Postman Environments, you can declare variables that are scoped to just the environment.  So you can make the url of your development and production server a variable and switch between environments to test the same APIs on different endpoints without changing the project.

Let's try this.  I create an environment by clicking the gear icon in the top-right and selecting `Manage Environments`.

![](/assets/posts/postman_3.png)

I select `Add` and add an environment called `Dev`.  I declare a variable `url` and its value is the location of my development server.

![](/assets/posts/postman_4.png)

Repeat for the `Production` environment.  You can see the same variable `url` but with a public IP associated with the `url` instead.

![](/assets/posts/postman_5.png)

Now I can use the variable in the request with {% raw %} `{{url}}` {% endraw %}.  I switch environments at the top-right. 

![](/assets/posts/postman_6.png)

Awesome.  You only have to maintain one project, and you can switch your environments easily to test your APIs if they are hosted in different places.  Keep in mind you can also share your environments in `Manage Environments` along with your projects that use variables.