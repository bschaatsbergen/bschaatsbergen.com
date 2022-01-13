---
title: "Agile Shower Thoughts"
date: 2021-10-03T14:29:15+01:00
tags: ["Agile"]
description: There's this ever growing need for speed mentality within software development. Everything needs to happen faster, but be aware of the risks. I jotted down some of my shower-thoughts to help you achieve speed.
---

Right of the bat I want to point out that this article isn't directly applicable to start-ups, where things have to be built from the ground up. But where there is steadiness and stability, a mindset that prioritizes shipping features with urgency can cause a lot of problems down the road. I think that software delivery is preferably a 3 step process that does not involve rushing features to market. Lets talk about that, and how we can iterate faster and be more efficient. This article is a tribute to a colleague that's unfortunately leaving my team, we carefully maintained a synergy, when it came to delivering software, we had it our way and it worked out great.

{{% tweet 1408717228296609793 %}}

Take a good look at this tweet and read it again, and again. Jonathan Smart changed part of my view on delivering software after reading ["Sooner Safer Happier"](https://www.amazon.com/Sooner-Safer-Happier-Patterns-Antipatterns/dp/1942788916/ref=sr_1_1?dchild=1&keywords=sooner+safer+happier&qid=1592332098&sr=8-1). To emphasize the above tweet I would like to cite a part of 'Pattern 7.1 - Go Slower to Go Faster'.

> "High-Performing organizations recognize that they need to go slower to go faster. In the same way as training for phsyical endurance events, such as a marathon or a cyclo-sportive, the recovery days are just as important as the training days, as that is when the body adapts and becomes stronger. Failing to schedule in enough recovery leads to overtraining, fatigue and eventually illness or injury. A relentless focus on going faster incurs technical debt, which leads to going slower."

# 3 step iteration

Taking the time to learn and overcome problems we encountered in a previous release is vital to continual improvement. I earlier mentioned that I think the delivery of software is a 3 step process, what I meant is as following:

- Learn: identify the problems and solve previous problems.
- Solve: design solutions in order to implement and solve the problems.
- Deliver: implement and deliver the solution.

![Learn Solve Deliver](/learn-solve-deliver.png)

Learning and solving are key, it shifts the way of thinking and viewing. Keeping the time between identifying the solution and delivering the solution minimal by automating as much of the delivery phase we allow ourselves to put the other 2 phases in the spotlight. The delivery should rather be unexciting, clear, automated and happen fast, very fast. This is enabled by adopting automation best practices, evolving out-of-the-box infrastructure capabilities and continuous learning. We need to lay focus on producing greater output from the engineers, as that's of most value to the entire team and the market.

![Learn & Solve more](/learn-and-solve-more.png)

The team has one direction to go about: identifty and solve problems. There's a phase dedicated to this, where we map ideas and experience to plausible solutions, the Solve phase. We focus on creating a shared vision through strategic thinking, what do we want and how do we go about it.

Btw, it's important to note that we should not bring all kinds of ideas in per iteration, nor should we try to capture all the things that could be improved. That's not the intention of the Learning phase, it will blur our vision. I think that it's better to stop noting down golden ideas along with every possible improvement or complaint per iteration. We'll end up doing the undoable and quickly lose focus. Improving is doing, get it done.

Focus on what's needed to improve the iteration and only the iteration. It should preferably have to do with the previous iteration and the team must commit to the improvement(s) as part of the Learning phase. This might mean that some capacity lies unused at some times, but the impact of delivery is not diminished. Try to turn this time into valuable time, give it to the engineers.

# Embrace continuous learning

Engineering bandwidth that is not fully occupied at all times is not a bad thing. Teams should take the time to pay off tech debt, having the ability to learn, improving the delivery pipeline (this is very important as we don't want to waste too much time here, remember?), writing documentation and refactoring the code base. Unfortunately this is often perceived as wasteful as it's of very little value to the business, which is busy fighting the battle of marketeers. Teams will then put in irrelevant busywork in the name of agility.

![Battlefield](/battlefield.png)

To achieve the operational excellence you're looking forward to it's again vital to keep learning. Sorry, I just can't stress it enough. Having the ability to share knowledge, learn and improve in the Learning phase makes the Solve and Delivery phase "just work out". This phase is what turns good teams into great teams and perfectly-oiled machines.

_Tldr_; unoccupied time of engineers should not be filled with meaningless busywork in the name of agility. Improve or innovate.

# Break down work into smaller units

Turning entire features into small isolated pieces of code making up sub features allows engineers to turn commits into pull requests. In the Solve phase we should have the design meeting to go over the problem statement. The feature should be broken down and have well defined logical boundaries. Our goal is to create small fast moving pull requests.

While software engineering is already an art and science by itself it's a good practice to write your code in such a way to stay within the boundaries of your sub-feature. Commit often and create many pull requests, even if the sub-feature isn't implemented yet nor is it completely functional. Doing so will create more visibilty on the code you've written for yours peers and allows them to follow your thinking process. We want to release as often as possible and prevent large pull requests (those are red-flags<sup>1</sup>).

_Tldr_; break down features, design to release often and have small fast moving pull requests

# Build momentum

All of this learning should result in engineers producing greater output. This might be cleaner code, clear communication, etc. It's up to the team or the lead developer to capture this in a momentum. I generally mean to say that there should be an increase of forward motion. Our goal is to become faster per iteration, right? Does this mean the lead developer should be beating the drums<sup>2</sup> on a dragon boat to coordinate the power and rhythm of the rowers? Yes and no, this is a hard question to answer as you might argue that the team should not be in need of a dragon boat drummer but rather create a momentum together. This is a very subjective question and I recommend you to take this one to the team first.

![Dragon Boat](/dragon-boat.jpg)

_Tldr_; include every improvement you make in a forward motion.

# Deploy as frequently as you can

Writing code is like making art, by the end of the day the artist turns the piece of art into value by making it an economic proposition. No matter how clean your code is or how well designed your new microservice is. It means nothing to no-one until its deployed. Deploying more frequently is basically bringing your product to market more often, it's vital to our iteration and should be completely automated. It's about focus, risk mitigation and confidence.

There's a variety of factors that prevent teams from increasing their deployment frequency, lets jot a few down.

Coupling, if a change in a service affects another service there's a deployment dependency created. Give your engineers room to improve this and allow them to get rid of high coupling across services as part of the Learning or unoccupied<sup>3</sup> phase. There's a great book I often recommend engineers to read when willing to brush up on their domain driven skills. Having the ability to define domain boundaries is important to reduce distributed business logic. Domain Driven Design by Eric Evans

Deploying is not releasing, if you can only go to production after it's reviewed by multiple teams and approved, you're doing something drastically wrong. The time and money spend holding back your go to market is not worth it. Try to turn this around by introducing well defined boundaries, merge small pieces of code along with their respective unit tests that eventually add up and create the feature and introduce feature toggles. We should change our view on deployments, deployments are not taking features live, releases are.

Periodic reminder, without well architected infrastructure and a good pipeline this is going to be tough. We'll be spending a lot of time here. Introducing such measures will also add-up to the confidence of the team to deploy more often. Deploying requires confidence.

Confidence, confidence is of the essence when deploying. Great teams spend time and effort introducing new tools in the Learning phase to achieve operational excellence. Without proper tooling it's excruciating to bust bugs and solve outages. The same goes for unit testing, I recently got involved in a discussion about unit tests and automation tests and that there might be overlap between the two. Unit tests test units, a functionality that relies on a dependency (often). Mocking or stubbing that dependency is fine, but keep in mind that you'll never test the actual dependency. That's where automation comes in, which captures a unit its dependency. Invest time in regressions and test automation, on one condition. Make it lightweight and fast, run it by automated interval or during a build and or release.

Infrastructure, It's important that you treat your resources throughout your infrastructure as a repeatable unit. This allows you to decouple yourself from taking the responsibility of managing the state of the unit. If a unit goes down that's fine, we account for it by scaling up a new unit. If a new version is released to a unit it can be rolled-back (explore rollback capabilities, some are faster and more fault-tolerant than others). I always encourage engineers to try out containers as they are a great example of treating compute environments as units that work together but are also isolated and independent from one another, they carry out a task together. I plan on writing about running and releasing with containers, keep an eye out.

Talk, discuss and brainstorm. Take some time to figure out how to mitigate risks, account for outages and how to setup up your infrastructure so that deploying becomes a boring automated repetitive task.

# Innovation is already part of the process

As mentioned earlier, it's all about improving and becoming faster. Adapting new technologies, paying off tech debt and playing around is what creates continual improvement as part of our iteration. Improvement and speed are indispensable to each other. Not baking this into your iteration will quickly turn continual improvement into the Shiny Object Syndrome<sup>4</sup> as your engineers only get to look at improvement once in a while.

![Dragon Boat](/complexity-cartoon.jpeg)

The idea of continual improvement and innovating is that we ensure that the Deliver phase is unexcited but optimized to death.

_Tldr_; Don't offload this, you'll quickly develop a Shiny Object Syndrome and lose focus on what we're supposed to be doing, excel in our iteration.


# The price of misused time is high

Meetings are more disruptive than ever to developers as they are forced to break focus and switch context. They are in need of time to carry out the 3 step iteration as efficient as possible. We've already mentioned twice that software engineering is an art and science by itself, it needs uninterrupted time to research, observe and create solutions. Prevent pulling your engineers in meetings that are not directly related to their ongoing iteration. It's better to isolate them from topics causing friction in getting the envisioned solution to production.

# Footnotes

1. Pull-requests with a large diff are a sign of not being able to break down your code in small fast moving parts. It's conflicting with the mentality of the engineer that tries to move fast as he/she has to dive into the problem, proposed solution and implementation. Having to backwards engineer a pull request is not fun.

2. The drummer on the dragon boat team serves a vital role. They are the pulse of the team and ensure that there's forward motion. The best drummers are experienced paddles and can relate to the paddlers their pain. They know the rules and regulations and understand the technique carried about by the paddlers. A common perception is that the dragon boat drummer is to pump people up, but it's about maintaining the calm and focus of a team.

3. With 'Unoccupied' I refer to the time where engineers are not busy. During this time they should focus on continual improvement and learning.

4. The Shiny Object Syndrome is the tedency to follow a current trendy something, yet drop this the moment a new trend takes its place. It's not directly a bad habit, but it's not as efficient as having continuous learning as a change stream. If you don't give engineers the change to make continuous improvement as part of their iteration, they'll end up looking at improvements as 'fun-time' projects and it's often not in-line with improving the 3 step process and has nothing to do with becoming faster, this is more like a mini hackathon. What I'm trying to say is that you should prevent engineers to view improvements as statically-bound objects they get to work on for a small subset of time once in a while.

