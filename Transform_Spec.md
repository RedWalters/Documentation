# InfoTransform Wiki

##### NB The functionality of these functions assumes that each tab is being processed out to it's own dataframe to then be joined together in stage 2 transform. 
##### This also assumes when using the trace you are using the tab.name as the table name when storing each tab.

## Functions

### Main

#### Info Transform

<code>infoTransform</code> takes 3 arguments to function: tabName, tabTitle, and tabColumns. 
</br></br>
The tabName (string), as one might expect, is the unchanged name of the tab in the excel document being transformed. (must remain unchanged as this is used to determine if the tab is alrady present in the spec upon rerunning)
</br></br>
The tabTitle (string) is usually the title of the tab given at the top of each sheet, if there isn't one then this can be chosen by the DE as it is not used to determine if the tab spec is already present in the info.json
</br>
This must be given as a string so if taking the title from the tab use cellCont to get the string value (see 'Cell Contents')
</br></br>
Lastly, the tabColumns (list) is given as the list of columns in the output dataset, this must be given as the columns variable that is being used for the trace as any column value passed to this function which isnt in the trace columns list will cause an error.
</br></br>
The function will run through the list of column headers call to the trace function for the most recent stored variable for each column.
</br></br>
The function will check if the transformStage section of the info.json file exists, if not it will create one and add the spec. 
If it does then it will pull out the list of specs and check if the given tab already exists in the list, if it does then it will replace the existing spec, if not then it will append it.
</br></br>
This function also reads and writes to the info.json file so there is no need to read/write it manually.

</br></br>
This function should be used in the loop for each tab (I have it just before the list of databaker dimensions)

<pre><code>def infoTransform(tabName, tabTitle, tabColumns):

    dictList = []

    with open('info.json') as info:
        data = info.read()

    infoData = json.loads(data)

    columnInfo = {}

    for i in tabColumns:
        underI = i.replace(' ', '_')
        columnInfo[i] = getattr(getattr(trace, underI), 'var')

    dicti = {'name' : tabName,
             'title' : tabTitle,
             'columns' : columnInfo}

    if infoData.get('transform').get('transformStage') == None:
        infoData['transform']['transformStage'] = []
        dictList.append(dicti)
    else:
        dictList = infoData['transform']['transformStage']
        index = next((index for (index, d) in enumerate(dictList) if d["name"] == tabName), None)
        if index is None :
            dictList.append(dicti)
        else:
            dictList[index] = dicti

    infoData['transform']['transformStage'] = dictList

    with open('info.json', 'w') as info:
        info.write(json.dumps(infoData, indent=4).replace('null', '"Not Applicable"'))
</code></pre>

#### Info Comments

<code>infoComments</code> is very similar to infoTransform except its primary use is for appending post transform changes to the spec. 

For example, when changing a DATAMARKER <code>df = df.replace({'Marker' : {'*' : 'Statistical Disclosure Applied'}})</code>
<br>
You would then add the trace <code>trace.Marker("Change * DataMarker to 'Statistical Disclosure Applied'")</code>
<br>
By running this function at the end of each tab's post transform script you can add this to the spec in the following form:
<pre><code>"Marker": [
"Change * DataMarker to 'Statistical Disclosure Applied'"]</code></pre>

</br>
This function takes arguments: tabName, and tabColumns. 

For this function to work correctly each tab should be being transformed into its own dataframe and therefore the post transform changes should be tab by tab.
</br></br>
This is because this functions calls the trace function variables by tab name so there should be a table for each tab using the tabName as the trace table name.
</br></br>
As the columns variable that was used for the infoTransform will likely no longer match up with the tab currently being transformed, you can use <code>list(df.columns)</code> to get a list of the columns for the current dataframe.


<pre><code>def infoComments(tabName, tabColumns):

    with open('info.json') as info:
        data = info.read()

    infoData = json.loads(data)

    columnInfo = {}

    for i in tabColumns:
        comments = []
        underI = i.replace(' ', '_')
        for j in getattr(getattr(trace, underI), 'comments'):
            if j == []:
                continue
            else:
                comments.append(':'.join(str(j).split(':', 3)[3:])[:-2].strip().lstrip('\"').rstrip('\"'))
        columnInfo[i] = comments

    columnInfo = {key:val for key, val in columnInfo.items() if val != ""}
    columnInfo = {key:val for key, val in columnInfo.items() if val != []}

    dicti = {'name' : tabName,
             'columns' : columnInfo}

    dictList = infoData['transform']['transformStage']
    index = next((index for (index, d) in enumerate(dictList) if d["name"] == tabName), None)
    if index is None :
        print('Tab not found in Info.json')
    else:
        dictList[index]['postTransformNotes'] = dicti

    with open('info.json', 'w') as info:
        info.write(json.dumps(infoData, indent=4).replace('null', '"Not Applicable"'))
</code></pre>

#### Info Notes

<code>infoNotes</code> is used to append any notes from the tabs to info.json.

It assumes that all the notes are saved under one string such as:
<pre><code>notes = """
Size of care homes is determined by number of beds.
Cumulative numbers are counts since data was first reported and will include cases which are no longer active
'w/c 15th June 2020' Based on return of 966 of Scotland's 1,080 adult care homes
'w/c 22nd June 2020' Based on return of 987 of Scotland's 1,080 adult care homes
'w/c 29th June 2020' Based on return of 1,006 of Scotland's 1,080 adult care homes
"""</code></pre>

<pre><code>def infoNotes(notes):

    with open('info.json') as info:
        data = info.read()

    infoData = json.loads(data)

    infoData['transform']['Stage One Notes'] = notes

    with open('info.json', 'w') as info:
        info.write(json.dumps(infoData, indent=4).replace('null', '"Not Applicable"'))</code></pre>

#### Excel Range

<code>excelRange</code> takes a bag of cells created with databaker and returns the lowest cell value and the highest cell value in the following form:
<pre><code>'{' + lowx + lowy + '-' + highx + highy + '}'</code></pre>

This is a core function for the transform spec as it can be used in conjuntion with the trace functions as follows:

<pre><code>trace.Period('Values found in range: {}', var = excelRange(period))</code></pre>

Which can then represent the location of the data being extracted as <code>"Period": "{B6-B19}"</code>, for example

<pre><code>def excelRange(bag):
    xvalues = []
    yvalues = []
    for cell in bag:
        coordinate = cellLoc(cell)
        xvalues.append(''.join([i for i in coordinate if not i.isdigit()]))
        yvalues.append(int(''.join([i for i in coordinate if i.isdigit()])))
    high = 0
    low = 0
    for i in xvalues:
        if col2num(i) >= high:
            high = col2num(i)
        if low == 0:
            low = col2num(i)
        elif col2num(i) < low:
            low = col2num(i)
        highx = colnum_string(high)
        lowx = colnum_string(low)
    highy = str(max(yvalues))
    lowy = str(min(yvalues))
</code></pre>

#### Cell Contents

<code>cellCont</code> is similar to excelRange except that it takes only a single cell as an argument, this is useful for tab titles and tab ranging periods.

<pre><code>def cellCont(cell):
    return re.findall(r"'([^']*)'", str(cell))[0]
</code></pre>

### Supplementary

#### Column to Number

<code>col2num</code> is used by <code>excelRange</code>. This function takes in a excel column coordinate (such as AX) and returns a number value of the location (eg 50). 
</br>
This is to simplify the ranking of the cell values in the range.

<pre><code>def col2num(col):
    num = 0
    for c in col:
        if c in string.ascii_letters:
            num = num * 26 + (ord(c.upper()) - ord('A')) + 1
    return num</code></pre>

#### Cell Locate

<code>cellLoc</code> is used by <code>excelRange</code>. This function takes a single cell from a cell bag and returns the cell coordinate. 
</br>
This is used in conjunction with <code>col2num</code> to rank cell values in a range.

<pre><code>def cellLoc(cell):
    return right(str(cell), len(str(cell)) - 2).split(" ", 1)[0]</code></pre>

#### Column Number to String

<code>colnum_string</code> is used by <code>excelRange</code>. This function is the reverse of <code>col2num</code>, it takes in a number (eg 50) and returns the corresponding column value (eg AX)
</br>
This is used in conjunction with <code>col2num</code> to rank cell values in a range.

<pre><code>def colnum_string(n):
    string = ""
    while n > 0:
        n, remainder = divmod(n - 1, 26)
        string = chr(65 + remainder) + string
    return string</code></pre>

