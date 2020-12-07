---
layout: post
published: true
categories:
  - personal
mathjax: false
featured: false
comments: true
title: Automation Tools Detected
---
## A worth adversary

While using Automation Tools today, a website had the **audacity** to tell me they suspected me of using Automation Tools! A couple hours later I have gained the website's trust and am browsing automatically without hassle. 

I'm leaving out the website and the security vendor that was circumvented.

## Factors that this security vendor seems to be analyzing:
Window Size: Detecting the default ChromeDriver window size.
`options.add_argument("start-maximized")`

User Agent: 
```
user_agent = random.choice(valid_windows_user_agents)
ua_input = 'user-agent=' + user_agent
options.add_argument(ua_input)
```

Pace: Moving too fast. Consitency of action patterns too strong.
`time.sleep(randrange(19,35))` //19-35 seconds delay

Referrer: Chrome starts on a "blank" page. Arriving to the website from the "blank" page is inhuman (unless it's your Chrome landing page)
`driver.get("https://www.google.com/")` 
