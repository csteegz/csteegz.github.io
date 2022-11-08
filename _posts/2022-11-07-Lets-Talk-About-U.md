---
layout: post
title: Let's Talk About UMD
date: 2022-11-07
---

So back when I was in college, there was this website called Deadspin.

It was a good website.

It was about sports.

Except when it wasn't. 

I used to love Deadspin.

For a couple of years, Deadspin (a sports website) had a series of articles called ["The Deadspin 25"](https://deadspin.com/the-deadspin-25-1721757320).
It was a satire of the college football rankings, where anyone could vote for who would be the top teams instead of just coaches or media members. 
The first year, the University of Central Florida won, without competition (I've since been told a professor was responsible for this - anyone with knowledge's input is welcome.) 

The second year, as a senior in college, I had to make my attempt. 

I used all the serious web penenetration testing reverse engineering skills, learned from the some of the most expensive consultants in Columbia, MD (looked at the site dev tools in Chrome and Googled what came up)  - it turned out that the developer had all the code for the backend on Github.

With that, I realized I should use the other thing I'd learned from the consultants - covering your ass - and got permission before making my attempt at dominating the competition:

![FUCK CPAA]({{site.url}}/assets/snip.png)

So, code in hand, I wrote a python script (now lost to time, but likely some verion of `while True: requests.post()`), took my free Azure credits and ran several instances of it on a VM for 20 somthing days.
Eventually, I packed up my life and went off to Seattle for a tour of duty in the Redmond code mines.
That year I finished second again to the UCF guy and an [entertaining article](https://deadspin.com/deadspin-25-dont-be-fooled-maryland-still-sucks-1786684104) was written.

In 2017, settled into my new life I took a little less effort. 
Since no Azure credits were at hand, my script would be run on my personal desktop when I wasn't otherwise occupied. 
Maryland showed up again in the rankings at #10 after [embarsassing UT.](https://deadspin.com/the-pieces-are-slowly-coming-together-for-maryland-1807524131)
Interestingly, the author described the coach DJ Durkin as `Not a Dick`. 

In 2018, it turned out they'd decided they didn't want to use their own voting software. 
Since my old script wouldn't work, it was time to rewrite my previous attemps and get Maryland to number 1!
My work team was using Kubernetes for deployments, so I wanted to play around with this cool new systems language, Go.
For completeness sake, I'll add a quick code sample now, to justify to myself that this is a tech blog and not just story time. 

<details>
<summary>Code Sample</summary>

{% highlight golang %}
func main() {
	start := time.Now()
	time_lapsed := make(chan time.Duration, req_count)
	school_channel := make(chan []choices, req_count)
	errs := make(chan error, req_count)
	tokens := make(chan form, req_count)
	throttle := time.Tick(rate)

	for w:= 0; w<thread_conunt; w++{
		go create_and_score(tokens, errs, time_lapsed,throttle)
		go create_entries(school_channel, errs, tokens,throttle)
	}
	for j := 0; j < req_count ; j++{
		schools := create_list()
		school_channel <- schools
		for _,school := range schools{
			c := global.GetCounter(school)
			c.Increment()
		}
	}
	close(school_channel)
	errors := 0
	elapsed := time.Duration(0)
	for i := 0; i<req_count; i++{
		select{
		case err := <- errs:
			errors++
			fmt.Println(err)
		case duration := <- time_lapsed:
			elapsed += duration
		}
	}
	count := req_count - errors
	fmt.Printf("%d requests succeeded %d failed, taking an average of %d ms\n",count,errors,
		(elapsed.Nanoseconds()/int64(count))/1000000)
	fmt.Printf("took a total of %s", time.Since(start).String())
	}
{% endhighlight %}

</details>

The third party service they were using had a rate limit and I, of course, respected it, voting only 10 times per second (per thread).
After I finished it near the end of July, I ran my software on my home desktop day and night, racking up votes for my favorite football teams. 
One by one the rankings came out, confirming that I had in fact properly stuffed the ballot boxes.
Surely this would be the year!

On my birthday, the [piece putting UMD at #2](https://deadspin.com/maryland-shouldnt-be-playing-football-this-weekend-1828743119) was published. 
Of course, it wasn't a celebration of the team's potential performance, or a light-hearted roasting of the fact that the readers made them write about a crappy team.
It was a brutal takedown of a coach and team that had negligently killed a player by forcing them to run without water during the DC June heat.
I don't remember how I felt about the article at the time. 
I don't remember if I thought about Jordan McNair's death, and considered stuffing the ballot box for a different team.
I wonder how the author felt; if he remembered that Maryland was up there with UCF as one of the schools that got more votes then it deserved; if he thought it was just a sick joke or if he was excited to write about the sick cynicism of college sports. 

Anyway, that was the last year of the Deadspin 25. 
I don't really know why it didn't return the next year.
At the time, I arrogantly thought I had killed it, forcing the writer to confront the realities of college football for the sake of a petty, possibly unreciprocated rivalry with an unknown adversary from UCF.
I was upset I would never get the chance to beat them.
I'm still disappointed about that, 4 years later, writing this blog. 

In 2019, Deadspin and some other sites were sold to a private equity firm. 
A bit after that, the Editor in Chief was fired, and the remainder of the staff resigned in disgust after being told to "stick to sports", even though the esoteric topic selection was what lent the site much of it's charm. 

Deadspin was a good website, killed by some buyers who didn't understand it. 

Doesn't it suck when that happens?