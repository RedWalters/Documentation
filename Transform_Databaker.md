# Transform Wiki

##### This contains template for a basic transformation.
##### ADD LINKS TO SPECIFIC FUNCTION DOCUMENTATION (SUCH AS DATABAKER AND SUCH WHEN THEYRE CREATED/UPDATED)

### Main

The first step in any transformation is scraping the distribution from the landing page given in the info.json specification.
Depending on the data publisher there may or may not be a scraper prepared for the landing page, or it may be out of date. If this is the case see 'Common Workarounds' (PLACEHOLDERNAME)

<pre><code>scraper = Scraper(linkToData)
scraper

scraper.select_dataset(title=lambda t: 'Expected Title' in t)
scraper
</code></pre>

Optionally you can then display the meta data for the chosen dataset, which can be useful for double checking you have the correct one

<pre><code>distribution = scraper.distributions[0]
display(distribution)
</code></pre>

Next is creating a list of all the tabs to transform

<pre><code>tabs = { tab: tab for tab in distribution.as_databaker() }</code></pre>

To filter out tabs that aren't relevant (such as contents) you can use [list comprehension](https://www.programiz.com/python-programming/list-comprehension)

<pre><code># only tabs where the name ends with "data"
tabs = [x for x in tabs if x.name.endswith("data")]

# only tabs where the name start with "data"
tabs = [x for x in tabs if x.name.startsswith("data")]
</code></pre>

We then move on to the main transformation of data. For detailed information on the variety of functions used in the transformation see the [Databaker Walkthrough](https://github.com/GSS-Cogs/databaker-walkthrough)

![Databaker before](https://github.com/RedWalters/Documentation/blob/master/resources/before.PNG?raw=true)

<pre><code>tidied_sheets = []

for tab in tabs:

    pivot = tab.filter('Beatles')

    groups = pivot.expand(DOWN).is_not_blank()

    members = pivot.shift(RIGHT).expand(DOWN).is_not_blank()

    assets = pivot.shift(2, -2).expand(RIGHT).is_not_blank()

    observations = members.fill(RIGHT).is_not_blank()

    dimensions = [
        HDim(groups, 'Groups', CLOSEST, ABOVE),
        HDim(members, 'Group Members', DIRECTLY, LEFT),
        HDim(assets, 'Personal Assets', DIRECTLY, ABOVE)
    ]

    tidy_sheet = ConversionSegment(tab, dimensions, observations)
    savepreviewhtml(tidy_sheet, fname="Preview.html")

    tidied_sheets.append(tidy_sheet.topandas())</code></pre>
    
Below you can see the output 'tidy' dataset and the preview which is created during the transformation.

Datasets which have multiple tabs will either have the tabs combined into one dataset, or will have multiple outputs for each. 
This depends on the complexity of the dataset.

  ![Databaker after](https://github.com/RedWalters/Documentation/blob/master/resources/after.PNG?raw=true)
