---
layout: post
title:  "Vimdoku – Sudoku with vim bindings"
shortdescription: >
    After finishing a fairly large software project as an assignment in school, I decided to work on my own project. I settled for creating a simple sudoku game with vim-bindings.
tags:   projects vimdoku
---
### Why?
The last reading period for my first year as a software engineering student at Chalmers University of Technology consisted mostly of a single, big project assignment. The assignment was basically to create an application of some sorts, using a few pre-decided software development methodologies. Since a lot of our education so far (and perhaps in future semesters as well) is based around Java, this project should as well be in Java.

The methodologies we were given were not very strict, but still defined some rules that we should follow. First of all, we were given one version of agile development, working in "iterations", where the first iteration was to get some kind of hard-coded prototype working, and then refine the application in further iterations. Furthermore we were to write a RAD document, write use cases for the application, create UML and sequence diagrams, and so forth.

The application we created was a statistics application for a drinking game called caps, which is very popular among students at Chalmers. The explicit goal for the application was not to be a complete solution -- the project spanned over 8 weeks and we would not have the time for that -- but instead be a solid foundation to develop further upon. The application we turned in can be found on my friend's GitHub: [hjorthjort/CapStat][capstat-gh].

It was a fun project, but I felt like a little too much focus was given to "paperwork", such as writing use cases. I personally felt like while they were a pretty good tool in the beginning of the project, where we decided what the application would do and not do, but after that they just became boring and cumbersome.

So, during the second half or so of the project, I felt like I wanted to take on a software project on my own, but be a little more "freestyle" and not rely as heavily on RAD and use cases.

### What?
I decided to create a **sudoku game** in Java, which would support a few **vim-like bindings** to navigate and play the game. There are a couple of reasons for this:

*   The most fantastic part about vim for me was when I finally started to "get" the keybindings. Nowadays, using HJKL feels infinitely more natural than using the arrow keys, in almost any given situation.
*   I love mathematics and algorithms, and when I get to combine them, I almost feel a little giddy inside. Sudoku felt like the perfect fit for this.
*   I wanted to expand my knowledge about GUI and frontend development. Previously, I have mostly been doing backend development (e.g. in CapStat, where I never even touched the frontend code) which I love, but I figured that having a diverse experience of software development is very valuable.

### How?
After deciding on what the application should be, I refined it and defined some guidelines -- both for the application and for the project itself:

*   The application should be a standalone Java application, runnable as a single `jar` file.
*   The application will not use networking of any sort: usage will only be local.
*   The entire application should be navigable using only the keyboard, using vim-like bindings such as HJKL.
*   Write the entire underlying functionality first, and only when that is finished begin developing the GUI.
*   JavaFX will be used as the GUI library.
*   Spend a lot of time on making the GUI look good, and try to do as little "ugly hacks" in the GUI code as possible.

Furthermore, I wanted this project to be an extended exercise in using git effectively. Even though I will be a one-man team, I will try to adhere to the popular [git flow workflow][git-flow] to an as large extent as possible.

### Summary
So the summary of the whole deal is that I wanted to take on writing an entire application myself, without using too much software development methodology. The application will be a simple sudoku game with vim-like bindings. The source code will be available at my GitHub repository: [Saser/vimdoku][vimdoku-gh].

During the development process, I will write a couple of blog posts detailing the development. I will be focusing on discussing my design choices for the code, and I will aim to be as pedagogic as possible. While I simply enjoy writing for myself, I also want the blog posts to perhaps be a resource for others that want to work on software projects themselves, so that they can learn from both my successes and my mistakes.

[capstat-gh]:   https://github.com/hjorthjort/CapStat
[git-flow]:     http://nvie.com/posts/a-successful-git-branching-model/
[vimdoku-gh]:   https://github.com/Saser/vimdoku
