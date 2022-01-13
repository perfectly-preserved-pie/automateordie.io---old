# The Last Words Project

## The Website
[I made a website for my new project!](https://lastwords.fyi) It's called Last Words.

## The Idea
Back in 2014 or so, I found out that the Texas Department of Corrective Justice posts every single executed inmate's last statement [on their website](https://www.tdcj.texas.gov/death_row/dr_executed_offenders.html). While definitely morbidly interesting, I also thought it would make a great coding project: what if I could automatically scrape each last statement along with the inmate's details and post them to a website? Furthermore, what if the website was *actually pretty*? I got my design inspiration from another Tumblr blog I had been following at the time, [A Sea of Quotes](https://www.aseaofquotes.com/). I wanted to apply that same look to these last statements.

## The Problem(s)
Well, it turns out that
1. Coding is fucking hard.
2. Web design is fucking hard.
  
And for the next 8 years or so, I floundered. I would get a burst of inspiration, work on the project, run into some coding obstacle, and then give up. 

![Rinse and repeat for years and years](https://i.imgur.com/qE2xj7x.jpg)

But it was always in the back of my mind, nagging me. If I could just get the code right, I could do it.

### Until now...
I'm not sure what changed, but I think it was a combination of boredom, winter break, being sick and isolated, and my decent-enough PowerShell skills. That gave me enough confidence (and dare I say, the skill?!) to start the project again and this time **stick with it**.

## The Stack
My original intent was to use a combination of an AWS LightSail WordPress instance (for the website) and PowerShell (to scrape the TDCJ webpage). However, I didn't like many of the WordPress themes and found the whole concept just *too overwhelming*. So many options, themes, addons, blocks, etc. I wanted to keep it simple.
Additionally, I found that webscraping with PowerShell is pretty difficult; there weren't many HTML modules (unless you want to get into .NET) I could use. I'd like to think I'm pretty good with PS but this project was just so far out of my wheelhouse I was at a loss.

So I decided to use Python, which meant learning it from scratch. That was tough; coming from PowerShell, it was terribly confusing and foreign. But I had heard about modules like BeautifulSoup and Pandas which seemed perfect for my use case. Through a lot of copy and pasting from Reddit & StackOverflow (as is tradition), I managed to get a working proof-of-concept which gave me the hope and motivation I needed to continue on.

For the website, I tried out Wix and SquareSpace, but again was overwhelmed by so many options. Also, I couldn't see how to create a layout that was optimized for quotes only instead of photos or long blog posts. So I went back to my source of inspiration, A Sea of Quotes. And then it hit me - why not just use Tumblr? I already had an excellent example of a quote post theme done right. Plus, Tumblr had a pretty good API with [a Python client available](https://github.com/tumblr/pytumblr).

## The Theme
Initially I started out with [Notations](https://www.tumblr.com/theme/8631) but thought it looked a little outdated. I wasn't satisified and kept looking.
After some further Googling, I stumbled upon [Zephyr](https://shoseii.tumblr.com/post/174988950529/zephyr-theme-14-live-preview-codes). The multi-column layout and infinite scrolling features were especially attractive. And the quote posts looked pretty clean as well.

## The Work
I had my work cut out for me. While [the base of the code had already been written for me](https://stackoverflow.com/a/64873079) (thank you, internet!), I still needed to do a number of additional things to make it so it would "fit" my new Tumblr blog. These things included:
* Using BeautifulSoup to grab specific links in an HTML table based on their tag position (in this case, the second "a href" tag)
 ```
   # Use bs4 to get the Offender Information URLs
   offender_information_urls = []
   # Select the correct <a href> tag based on second tag I guess
   for link in soup.select('tr>td:nth-child(2)>a'):
       offender_information_urls.append(f"{base_url}/" + link['href'])
   # Add the URLs to the dataframe
   df["Offender Information"] = offender_information_urls
  ```
* Removing all inmates that didn't have a last statement or declined to say one
 ```
# Remove all inmates that don't have a last statement
# We'll first create a list of keywords indicating no last statement
# https://stackoverflow.com/a/43399866
keywords = ['This inmate declined to make a last statement.','No statement was made.','No statement given.','None','None.','None ','(Written statement)','Spoken: No','Spoken: No.','No','No last statement.','No, I have no final statement.', '']
empty_statements = df[df['Last Statement'].isin(keywords)].Execution.count() + df['Last Statement'].isnull().sum().sum()
# Drop all rows containing these "no last statement" keywords
df = df[~df['Last Statement'].isin(keywords)]
# Now we drop all rows containing NaN
# https://hackersandslackers.com/pandas-dataframe-drop/
df.dropna(axis=0,how='any',subset=['Last Statement'],inplace=True)
 ```

* Working around Tumblr's API limits
* Iterating over batches of dataframes
* Using matplotlib to generate simple statistical plots of the data
 ```
 # Age distribution
# https://riptutorial.com/pandas/example/5965/grouping-numbers
age_groups = pd.cut(df.Age, bins=[18, 20, 29, 39, 49, 59, 69, 79, 89], labels=['18 to 20 years old', '21 to 29 years old', '30 to 39 years old', '40 to 49 years old', '50 to 59 years old', '60 to 69 years old', '70 to 79 years old', '80 to 89 years old'])
# Plot the groups
# https://stackoverflow.com/a/40314011
age_groups_count = df.groupby(age_groups)['Age'].count()
age_plot = age_groups_count.plot(kind='bar', title='Age Distribution of Executed Inmates in Texas, 1982-2021', ylabel='Number of Inmates', xlabel='Age Group')
# Annotate the bars
# https://stackoverflow.com/a/67561982
age_plot.bar_label(age_plot.containers[0], label_type='edge')
# Save the plot as a PNG
print("Saving the plotted age graph...")
plt.savefig("/tmp/age_distribution.png", bbox_inches = 'tight')
# Close the figure window to prevent the next graph from using the same values
# https://stackoverflow.com/a/8228808
plt.clf()
```
* Using SymPy to *solve an algebraic equation* (woah! Haven't done that in a while!)
```
# Algebra comes in handy here. My high school math teacher has finally been vindicated 15 years later 👏
# It doesn't matter how many rows we have: we need to post x posts until we remain with 300 posts
# The last 300 posts will be queued
# Use SymPy to solve an algebraic equation
# https://scipy-lectures.org/packages/sympy.html#equation-solving
x = Symbol('x')
# Solve for x: how many posts do we need to immediately publish?
posts_to_publish = int(solve(len(df.index)-300-x, x)[0])
```

## The Journey
I've learned a ton about Pandas, dataframes, and Python in general. Even though the base of my code was simply copied & pasted, the work I did have to do was enough to send me on a Python journey and as a result I feel a little more confident in my coding skills. I have a much better handle on the various Python modules and I'm honestly in awe of the whole Python module ecosystem. It's amazing. There's a module for damn near everything. Pandas, Numpy, and BeautifulSoup have been an incredible help here.

Also, web scraping is _tough_. There's almost no coordination or standards from one page to the next; I spent so much time trying to account for every little deviation I was almost driven mad.

![So many edge cases...](https://imgs.xkcd.com/comics/data_pipeline.png)

With all that being said, it was fun! And incredibly frustrating at times! 🌈

## The Code
[I've put up my code on my GitHub](https://github.com/perfectly-preserved-pie/lastwords). Feel free to flame me for my awful spaghetti code. I won't lie: I definitely coded with the intention of just making it work rather than making it work *well*.

I can finally close the 122 tabs I have open related to this project. Jesus.
