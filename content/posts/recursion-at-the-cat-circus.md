---
title: "Recursion at the Cat Circus"
date: 2021-07-16T16:50:24Z
draft: true
---
### Some Background

Recently at work I faced an interesting challenge while implementing a new feature. When users load up a new instance of our app, the app scans
their input data and then generates a series of default values that are stored in a `state` object. Elements like height, color, order, and labeling
are all pieces of state that can be updated, saved, imported, and exported. Users can also supply their own state data to be used in lieu of the defaults, 
and users can have any number of different states associated with each project. The challenge - allow users to supply incomplete state data, and have the app 
fill in the blanks. For the solution, let's head to the cat circus!
![cat circus](https://assets.dnainfo.com/photo/2016/9/1473349372-272315/extralarge.jpg) 

### The Scenario

The cat circus is in town, and you and a friend managed to snag tickets to opening night! You both arrive early and take your seats, notepads in hand - ready to 
log everything you see during the show. But there's one thing neither of you accounted for - these cats are fast!

![fast cat](https://33.media.tumblr.com/2af266dbaf88f82a533e727b7d3ca783/tumblr_nvwvmmEucR1uuyy36o1_500.gif)

So fast and mesmerizing is the action that you're unable to jot it all down in time. Each time you look up from your notebook, a brand new cat with a brand new skill 
has been introduced, and you're barely keeping up. After the final act ends and all the cats come out to take a bow, you take a look at your notes and realized you may
have missed some action. 

```
let catCircus = {
    actOne : {
        01 : {
            name : 'Holly', 
            age : 7,
            act : 'spun a plate on her head'
        },
        02 : {
            age : 2, 
            act : 'can write in cursive'
        },
        03 : {},
        04 : {
            name : 'Mr. Shoeshine',
        },
    },
    actTwo : {
        ...continued
    },
}
```
You glance over to your friend who looks up from their own notebook, similarly crestfallen. The bad news - you were both suckered in by the wildly cute and entertaining cats, 
and your notes reflect your distraction. The good news - between the two of you, you're confident you've captured a complete dataset. You run out of the auditorium (after buying a shirt)
and head straight to your computer. You're going to write an algorithm that takes in both of your incomplete datasets and spits out a complete one. And since the dog frisbee throw competition 
is in town next week you're going to make sure your algorithm is extensible, so you can kick back and enjoy those frisbee pups. 