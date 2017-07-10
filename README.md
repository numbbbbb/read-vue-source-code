# Read Vue Source Code

## Why

Imagine you find a new framework, let's say it's Vue. You Google the tutorials, build a HelloWorld project and start to use it in company projects.

Now you are familiar with that framework, what should you do next?

There are several choices. But for me, I just want to see what's beneath the surface.

How to implement the two-way binding?

How to design the lifecycle and what's the right time to call hooks?

How to build such a big project?

What's the right way to test it?

Why or why not to have that feature?

In order to answer those questions, I decided to read the source code. The best way to learn is to teach, so I write this series to show you how I read the code and what I gain from it.

## How to Read This Series

You can sit here and read, but I highly recommend you to try it by yourself. You need to get your hands dirty to really learn something.

I will record the actions I take, like **download source code from xxx** and **jump to file xxx and find xxx**. You can replay them by yourself.

In this series, I won't try to explain everything. My purpose is to help you understand Vue's structure and how it works. I will leave some functions or files for you to read in practice sections.

If you have any questions and advice, feel free to contact with me! You can use issue as comment, or email me at *lj925184928@gmail.com*.

## Who This Series is For

Everyone who is familiar with web frontend development can read this series. 

If this is your first time reading source code, you can learn how to read them. 

If you are a Vue user you can understand the daily used framework well. 

If you have read the source code by yourself before, you can read it again from my point of view.

Exciting? Here we go!

## Contents

- [Find the Entry]()
- Dig into the Core
- Initialization
- Dynamic Data - Observer, Dep and Watcher
- Dynamic Data - Lazy, Sync and Queue
- View Rendering - Intruduction
- View Rendering - Compiler
- View Rendering - Patch
- Conclusion

## License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

