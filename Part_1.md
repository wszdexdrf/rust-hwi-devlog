## Acknowledgement
I really appreciate the bitcoin seminar going alongside the first portion of the project. At the beginning I thought this was time I could have spent actually working on the project, but then later I actually came across stuff which I studied as part of the seminar. If not for it, I would have tacked on some half thought approach, which would have not been good for the long term. I also reall want to thank my mentor. Whenever I got into any sort of problem (which was often), she would help me.

## 0 BCE
I looked into the source code for rust-hwi. It is much simpler than I thought. It is just using commands to call the hwi exectuable (from bitcoin-core) and passing the arguements. I looked at the changelog from the last supported version of hwi by rust-hwi and made necessary changes so that rust-hwi works with the latest executable of hwi. I also added commits to solve few small "good-first-issue" issues for rust-hwi.

## Week 1
Now I know there is a set structure for commits and Pull Requests (this is my first time working on a public git project). I made all the necessary changes pointed out by my mentor and rebase almost a hundred times. 

## Week 2
I finished working on adding Continuous Integration for rust-hwi. Now testing would be much easier and PR would be merged much quicker.

## Week 3
Made miscellaneous changes to fix some warnings. Also changed CI to fail instead of warn on warnings. I started preparing for moving rust-hwi to PyO3.

## Week 4
Created PR for moving to PyO3. PyO3 is library which helps us call Python code directly from rust, basically helps us talk to the Python interpretor. With this, instead of the previous method of calling the hwi executable and passing arguements through commands, now we can simply load the Python library (hwilib) into memory and call the required functions directly. It also gives better access to Python exceptiions which helps in debugging.

I also came accross a situation where the tests would pass for trezor emulator but fail for ledger devices, even though this was a problem in the HWI side (which I came to know about a bit late). I have to add more emulator tests in the CI to know about problems quicker.

## Week 5
I made some refactoring to the code so that it is more aligned to working with PyO3. Also I abstracted away some of the "innards" of PyO3 and related stuff. It isn't tidy inside 😅. I started working on implementing remaining functions not implemented in rust-hwi. It is not really difficult as I simply have to add a function of similar name and then pass all the arguements via PyO3.

## Week 6
I came across a hitch. Somehow I am not able to get some functions to not work even with the native hwi executable. I quickly finished the rest of the functions and postponed the rest. I fell sick in between, when my mentor pull up the slack and push some commits to rust-hwi. 🙇

## Week 7
I start working on some bitcoindevkit/bdk issues, as the last (and major) part of my project is to integrate rust-hwi with bdk. I also created a PR for automating the installation of hwi and related dependecies.