---
title: "Roll Your Own (Ish)"
date: 2021-07-28T01:28:56Z
draft: false
---
## I can do less.. more!

One of the greatest joys I've experienced since learning to program has been the newly 
gained ability to reimagine how I interact with technology. A lot of my interactions through 
an internet browser or phone app are rooted in routine and familiarity. Ever look at a friend's phone home screen? It's always unfamiliar and _the worst_. 
!['bad phone'](https://www.iphonehacks.com/wp-content/uploads/2017/06/iOS-10-vs-iOS-11-home-screen.jpeg)

When I want to know how much a copy of StarFox for Super Nintendo is going for, I hit `command + space` in MacOS to bring up Spotlight, type 'ebay', hit `enter` to open Chrome and navigate to ebay.com, I hit `tab` to focus the search text input, type 'sonic the hedgehog super nintendo', hit `arrow down` when the autocomplete loads to navigate to the recommended search, hit `enter`, and on. 

I do this a lot. Like all the time. And I don't even think about it. I'm sure there are a half dozen smarter ways to do this, I just have no interest in finding out and implementing whatever they are. 
Some of the most difficult 'quality of life' improvements then, are in areas where the addage 'if it ain't broke, don't fix it' reigns supreme. More and more these days I find that I'm willing to break something 
that was working 'just fine' in the pursuit of incremental improvement. I alias the terminal commands I use most, I weigh my coffee grounds in the morning for a better brew - sometimes I'll even write code to solve problems
that aren't actually problems. Let's go. 
!['weighing coffee'](https://www.homegrounds.co/wp-content/uploads/2019/12/wade-austin-ellis-CCofpiZeMbs-unsplash.jpg)

## The scenario

I collect vinyl records. Not for any high-minded audiophile reasons, but mostly because I enjoy the physical interaction with music that vinyl allows for. I like supporting the artists, labels, pressing plants, and record stores that all contribute to the vinyl ecosystem. Even though Spotify accounts for the bulk of my music-listening, records are still the warm blanket that I can count on if the internet goes out, or if Spotify gets gobbled up by the Fox Corporation. 
!['some records'](https://i.imgur.com/47kA8ji.jpg)

Often while I'm sitting at my desk, the following scenario will play out 
- I'm listening to a playlist on Spotify
- I hear a song I like, and want to know if the parent album is available for purchase
- I open a new Chrome tab and navigate to __www.discogs.com__
- I search for the artist or album 
    - Sometimes this takes a couple tries, as generic titles may have multiple listings
- From the correct album page, I click through to the marketplace
    - If there are many listings, I need to filter format to 'vinyl'
- I sort price low-to-high
    - This frustratingly doesn't factor in shipping costs, which can often exceed the listing price of the album
- I take a mental note of my findings
- I then go about my day, happy in my routine

Recently this routine finally hit the critical mass of annoyance and inefficiency required for me to want to address it. Luckily discogs has a publicly accessible API, so I figured I could whip up a CLI tool that 
- Captures the currently playing track from Spotify
- Queries the discogs database for the associated album and returns the `master_id`
    - The `master_id` is important - there may be 50 different formats/pressings/color variants/reissues of an album, and we want to include them all 
- Using the `master_id`, queries the discogs marketplace for all vinyl copies of the album
- Returns the low/high/average __total__ (shipping inclusive) price of the album, along with a link to the album in the marketplace

Luckily, there are already a handful of useful packages that allow me to get up and running pretty quickly. 
- `spotify-node-applescript` provides a straightforward API to interact with a current Spotify session
- `node-fetch` allows for standard fetch syntax in a node app 
- `chalk` lets me make my CLI text _pretty_
- `minimist` brings simple arg parsing to the table

And away we go!
```js
const spotify = require('spotify-node-applescript');
const fetch = require("node-fetch");
var argv = require('minimist')(process.argv.slice(2));
const chalk = require('chalk')
const key = process.env.KEY
const secret = process.env.SECRET

// GLOBAL CHALK VAR
const error = chalk.bold.red
const friendly = chalk.bold.blue

if (argv.h || argv.help){
    console.log(friendly(`spotbought captures your currently playing spotify album, and then queries the discogs.com marketplace for available copies.
    use the '-c' flag to set currency value (ie USD, CAD, JPY). 
    `));
    return
}

spotify.getTrack(function(err, track){

    let [artist, album] = [track.artist, track.album]
    if(err){
        console.log(error("encountered an error :(", err));
    } else {
        console.log(friendly(`Querying discogs database for ${album}`));
        fetch(`https://api.discogs.com/database/search?artist=${artist}&release_title=${album}&key=${key}&secret=${secret}`)
            .then(resp => resp.json())
            .then(query => {
                if(query.results[0]){
                    console.log(friendly('querying discogs marketplace for available copies'))
                    fetch(`https://api.discogs.com/marketplace/stats/${query.results[0].master_id}?curr_abbr=${argv.c ? argv.c : 'USD'}`)
                        .then(resp => resp.json())
                        .then(stats => {
                            let outputStr; 
                            if(stats.num_for_sale > 0){
                                const URL = `https://www.discogs.com/sell/list?master_id=${release_id}&format=Vinyl`
                                outputStr = 
                                `There are currently ${stats.num_for_sale} copies of ${album} for sale, with a low price of ${stats.lowest_price.value} ${stats.lowest_price.currency}. \n 
                                Here is a link to the album's vinyl listings on Discogs' marketplace : ${URL}                  
                                `
                                console.log(friendly(outputStr));
                            } else {
                                outputStr = 
                                `Looks like there aren't any copies of ${album} for sale on the discogs marketplace :/`
                                console.log(error(outputStr))
                            }
                        })
                } else {
                    console.log(`looks like we couldnt find any listings for ${album} on discogs :(`);
                }
            })
    }

});
```
And pretty quickly, I was getting results. The deeply nested fetch calls could be refactored later - the important part was that things were working. The only issue?

### Discogs' marketplace queries are bullshit

Directly from Discogs' API documentation for the `release statistics` endpoint: 

> Retrieve marketplace statistics for the provided Release ID. These statistics reflect the state of the release in the marketplace currently, and include the number of items currently for sale, lowest listed price of any item for sale, and whether the item is blocked for sale in the marketplace.

Unfortunately, this endpoint gets a bunch of stuff wrong. In my limited usage,
- The `# of items currently for sale` value was incorrect 100% of the time
- The `lowest listed price` value did not include shipping costs, which made this value irrelevant someting like _half_ the time. 
- I don't get any other price-related insights except for the above

And most damning of all, 
- I'm not able to filter the query or the results by format!

This means that my query may return a low price of $2.50, even though that number could represent the lowest price for a CD - there may not even be any vinyl copies for sale! And given that the vast majority of vinyl albums have a CD or cassette counterpart, and that CDs are just about _always_ cheaper on average, it became clear that the data returned from my marketplace query was effectively useless. Woof!

## Rolling my own...ish

From the first iteration of building and testing, I was happy with how everything was working up until the final discogs API call - good bones, just needs a little renovation. But given that discogs' marketplace endpoint wasn't going to cut it, I needed a different approach. Enter scraping with Puppeteer!
!['scraping'](https://thumbs.gfycat.com/BarrenDisfiguredFieldmouse-size_restricted.gif)

Puppeteer is a 'Node library which provides a high-level API to control Chrome or Chromium over the DevTools Protocol'. Basically, Puppeteer allows for the automation of all the mundane steps and interactions that had become my mindless routine. Puppeteer's API is pretty straightforward, and in a few short lines of code can 
- open a new headless instance of the Chromium browser
- open a new page
- navigate to the discogs marketplace URL generated from our discogs database query
- sort the results by price via the dropdown select 
- query a specific html element, storing each value in an array 
- return that array back to my node app for processing 

Interestingly enough, the discogs marketplace actually __does__ list the total (shipping inclusive) price for each marketplace item 
!['total price'](https://i.imgur.com/afdmJsc.png)

And better yet, this total price value is nestled in an easily targetable span class!
!['target span'](https://i.imgur.com/yDiJjmI.png)

The puppeteer code looks like this
```js
async function puppeteerScrape(URL){
    const browser = await puppeteer.launch()
    const page = await browser.newPage()
    console.log(blue('navigating to discogs marketplace'));
    await page.goto(URL);
    await page.select('#sort_top', 'price,asc')
    console.log(blue('scraping prices from listings'));
    let pricesArr = await page.$$eval(".converted_price", elements=> elements.map(item=>item.innerText))
    await browser.close()
    console.log(green('success'));
    processPrices(pricesArr)
}
```

The returned array values are strings that look like `<span>about</span>             $21.94 <span>total</span>` or `            $15.44 <span>total</span>`, so a little bit of processing needs to happen 
next. Mapping over the returned array with a simple regex pattern will rip out the dollar amounts, which are converted to `floats` and pushed into a new array. A helper function `average()` returns the average
marketplace price, with the first and last elements of the sorted `strippedPrices` array represent the low and high prices, respectively. 
```js
function processPrices(priceArr){
    let strippedPrices = []
    let re = new RegExp('[0-9]{2,3}.[0-9]{2}')
    priceArr.map(price => {
        strippedPrices.push(parseFloat(re.exec(price)[0])) // parse dollar amount from string
    })
    const average = (array) => array.reduce((a, b) => a + b) / array.length;

    const lowPrice = strippedPrices.sort((a, b) => (a < b ? -1 : 1))[0];
    const averagePrice = average(strippedPrices)
    const highPice = strippedPrices.sort((a, b) => (a < b ? -1 : 1))[strippedPrices.length - 1]

    ... // render results
}

```
Just like that, I'm getting the information I came for!

## Wrapping up

When I talk to fellow junior developers and other folks looking to break into the field, we often talk about the 'repository graveyard' phenomenon - github accounts packed with repositories for half-finished ideas, personal projects
that never went anywhere, uDemy code-alongs that never got finished, and on. When we wax philosophical about the conundrum, the issue of motivation always looms large. For junior developers, the motivation may be to secure employment, 
get a bigger paycheck, or stave off the omnipresent inferiority complex for another day. 

Personally, I've found that my biggest motivator as a new developer is my desire to be well and truly __lazy__.
!['garf'](https://i.pinimg.com/originals/3e/8b/65/3e8b65ed6773bf11fea5d1330dbb0f2f.jpg)
If I can spend an afternoon learning how to program a solution that saves me 20 seconds every other day for the next 5 years, I'm going to do it. And if I can learn some new things in the process, even better. But if I can do all that, and then blog about it - well that's just the greatest gift of all : )  

[post-script - you can checkout the full project [here](https://github.com/matthewlawrenceklein/spotbought/blob/master/spotBought.js)]