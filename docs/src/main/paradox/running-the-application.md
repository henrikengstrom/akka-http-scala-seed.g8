Final testing
-----------------------
When you ran the example for the first time, you were able to create and retrieve multiple users. Now that you understand how the example is implemented, let's confirm that the rest of the functionality works. We want to verify that:

* If we try to retrieve users when none exist, we get an empty list.
* If we try to retrieve a specific user that doesn't exist, we get an informative message.
* We can delete users.

To test this functionality, follow these steps. If you need reminders on starting the app or sending requests, refer to the @ref:[instructions](index.md) in the beginning.

Comment: I thought it would be fun to have the user try these without hints. But, I put the hints in just in case. If you agree, please remove them.

1. If the Akka HTTP server is still running, stop and restart it.
1. With no users registered, use your tool of choice to:
    1. Retrieve a list of users. Hint: use the `GET` method and append `/users` to the URL.

    You should get back an empty list: `{"users":[]}`

    1. Try to retrieve a single user named `MrX`. Hint: use the `GET` method and append `user/MrX` to the URL.

    You should get back the message: `User MrX is not registered.`

    1. Try adding one or more users. Hint: use the `POST` method, append `/user` to the URL, and format the data in JSON, similar to: `{"name":"MrX","age":31,"countryOfResidence":"Canada"}`

    You should get back the message: `User MrX created.`

    1. Try deleting a user you just added. Hint: use the `DELETE`, and append `/user/<NAME>` to the URL.

    You should get back the message: `User MrX deleted.`  

Comment: If the student is a Scala user, they will be familiar with SBT, right? I would remove the following paragraph and section about the build file, unless there is something special about it that you want to teach them?

You can run the application from the command line or an IDE. The final topic in this guide describes how to run it from IntelliJ IDEA. However, before we get there, let’s take a quick look at the build tool: sbt.

## The build files

sbt uses a build.sbt file to handle the project. This project’s build.sbt file looks like this:

@@snip [build.sbt]($g8root$/build.sbt)


Now that you've confirmed all of the example functionality, see how simple it is to integrate the project into an IDE.
