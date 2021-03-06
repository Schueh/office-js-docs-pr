# Create custom functions in Excel (Preview)

Custom functions (similar to user-defined functions, or UDFs), enable developers to add any JavaScript function to Excel using an add-in. Users can then access custom functions like any other native function in Excel (such as `=SUM()`). This article explains how to create custom functions in Excel.

The following illustration shows you how an end user would insert a custom function into a cell. The function that adds 42 to a pair of numbers.

<img alt="custom functions" src="../images/custom-function.gif" width="579" height="383" />

Here’s the code for the same custom function.

```js
function add42(a, b) {
    return a + b + 42;
}
```

Custom functions are now available in preview. Follow these steps to try them:

1.  Install Office 2016 for Windows and join the [Office Insider](https://products.office.com/en-us/office-insider) program.
2.  Clone the [Excel-Custom-Functions](https://github.com/OfficeDev/Excel-Custom-Functions) repo and follow the instructions in the README.md to start the add-in in Excel.
3.  Type `=CONTOSO.ADD42(1,2)` into any cell, and press **Enter** to run the custom function.

See the **Known Issues** section at the end of this article, which includes current limitations of custom functions and will be updated over time.

## Learn the basics

In the cloned sample repo, you’ll see the following files:

- **customfunctions.js**, which contains the custom function code.
- **customfunctions.json**, which contains the registration JSON that tells Excel about your custom function. Registration makes your custom functions appear in the list of available functions displayed when a user types in a cell.
- **customfunctions.html**, which provides a &lt;Script&gt; reference to the JS file. This file does not display UI in Excel.
- **manifest.xml**, which tells Excel the location of the HTML, JavaScript, and JSON files; and also specifies a namespace for all the custom functions that are installed with the add-in.

### JavaScript file (customfunctions.js)

The following code in customfunctions.js declares a custom function called `add42` similar to the previous example, but it adds 42 to a single input instead of two.

```js
function add42(num) {
    return num + 42;
}
```

### JSON file (customfunctions.json)

The following code in customfunctions.json specifies the metadata for the same function.

> [!NOTE]
> Detailed reference for the registraion JSON of custom functions, including options not used in this example is at [Custom Functions Registration JSON](https://dev.office.com/reference/add-ins/custom-functions-json).

Note the following about the properties in this example:

- There's only one custom function, so there's only one member of the `functions` array.
- The `name` property defines the function name. As you see in the animated gif shown previously, a namespace (`CONTOSO`) is prepended to the function name in the Excel autocomplete menu. This prefix is defined in the add-in manifest, described below. The prefix and the function name are separated using a period, and by convention prefixes and function names are uppercase. To use your custom function, a user types the namespace followed by the function's name (`ADD42`) into a cell, in this case `=CONTOSO.ADD42`. The prefix is intended to be used as an identifier for your company or the add-in. 
- The `description` appears in the autocomplete menu in Excel.
- When the user requests help for a function, Excel opens a task pane and displays the web page found at the URL specified in `helpUrl`.
- The `result` property specifies the type of information returned by the function to Excel. The `type` child property can `"string"`, `"number"`, or `"boolean"`. The `dimensionality` property can be `scalar` or `matrix` (a two-dimensional array of values of the specified `type`.)
- The `parameters` array specifies, *in order*, the type of data in each parameter that is passed to the function. In this case there is only one. The `name` and `description` child properties are used in the Excel intellisense. The `type` and `dimensionality` child properties are identical to the child properties of the `result` property described above.
- The `options` property enables you to customize some aspects of how and when Excel executes the function. There is more information about these options later in this article.

 ```js
{
	"functions": [
		{
			"name": "ADD42", 
			"description":  "Adds 42 to the input number",
			"helpUrl": "http://dev.office.com",
			"result": {
				"type": "number",
				"dimensionality": "scalar"
			},
			"parameters": [
				{
					"name": "num",
					"description": "Number",
					"type": "number",
					"dimensionality": "scalar"
				}
			],
			"options": {
				"sync": true
			}
		}
    ]
}
```

> [!NOTE]
> The custom functions are registered when a user runs the add-in for the first time. After that, they are available, for that same user, in all workbooks (not only the one where the add-in ran initially.)

### Manifest file (manifest.xml)

The following is an example of the `<ExtensionPoint>` and `<Resources>` markup that you include in the add-in's manifest to enable Excel to run your functions. Note the following about this markup:

- The `<Script>` element and its corresponding resource ID specifies the location of the JavaScript file with your functions.
- The `<Page>` element and its corresponding resource ID specifies the location of the HTML page of your add-in. The HTML page includes a `<Script>` tag that loads the JavaScript file (customfunctions.js). The HTML page is a hidden page and is never displayed in the UI.
- The `<Metadata>` element and its corresponding resource ID specifies the location of the JSON file.
- A `<Namespace>` element and its corresponding resource ID specifies the prefix for all custom functions in the add-in.


```xml
<VersionOverrides xmlns="http://schemas.microsoft.com/office/taskpaneappversionoverrides" xsi:type="VersionOverridesV1\_0">
    <Hosts>
		<Host xsi:type="Workbook">
			<AllFormFactors>
				<ExtensionPoint xsi:type="CustomFunctions">
					<Script>
						<SourceLocation resid="residjs" />
					</Script>
					<Page>
						<SourceLocation resid="residhtml"/>
					</Page>
					<Metadata>
						<SourceLocation resid="residjson" />
					</Metadata>
					<Namespace resid="residNS" />
				</ExtensionPoint>
			</AllFormFactors>
		</Host>
	</Hosts>
	<Resources>
		<bt:Urls>
			<bt:Url id="residjson" DefaultValue="http://127.0.0.1:8080/customfunctions.json" />
			<bt:Url id="residjs" DefaultValue="http://127.0.0.1:8080/customfunctions.js" />
			<bt:Url id="residhtml" DefaultValue="http://127.0.0.1:8080/customfunctions.html" />
		</bt:Urls>
		<bt:ShortStrings>
			<bt:String id="residNS" DefaultValue="CONTOSO" />
		</bt:ShortStrings>
	</Resources>
</VersionOverrides>

```


## Asynchronous and synchronous functions

If your custom function retrieves data from the web, you need to make an asynchronous call to fetch it. When calling external web services, your custom function must:

1. Return a JavaScript Promise to Excel.
2. Make the HTTP request to call the external service.
3. Resolve the promise using the `setResult` callback that provided for you by Excel. `setResult` sends the value to Excel.

The following code shows an example of an asynchronous custom function that retrieves the temperature of a thermometer. Note that `sendWebRequest` is a hypothetical function, not specified here, that makes an AJAX call to a temperature web service.

```js
function getTemperature(thermometerID){
    return new OfficeExtension.Promise(function(setResult){
        sendWebRequest(thermometerID, function(data){
            setResult(data.temperature);
        });
    });
}
```

> [!NOTE]
> Custom functions are asynchronous by default. To designate functions as synchronous set the option `"sync": true` in the `options` property for the custom function in the registration JSON file.

## Streamed functions

An asynchronous function can be streamed. Streamed custom functions let you output data to cells repeatedly over time, without waiting for Excel or users to request recalculations. The following example is a custom function that adds a number to the result every second. Note the following about this code:

- Excel displays each new value automatically using the `setResult` callback.
- The final parameter, `caller`, is never specified in your registration code, and it does not display in the autocomplete menu to Excel users when they enter the function. It’s an object that contains a `setResult` callback function that’s used to pass data from the function to Excel to update the value of a cell.
- In order for Excel to pass the `setResult` function in the `caller` object, you must declare support for streaming during your function registration by setting the option `"stream": true` in the `options` property for the custom function in the registration JSON file.

```js
function incrementValue(increment, caller){
    var result = 0;
    setInterval(function(){
         result += increment;
         caller.setResult(result);
    }, 1000);
}
```

## Cancellation

You can cancel streamed functions and asynchronous functions. Canceling your function calls is important to reduce their bandwith consumption, working memory, and CPU load. Excel cancels function calls in the following situations:

- The user edits or deletes a cell that references the function.
- One of the arguments (inputs) for the function changes. In this case, a new function call is triggered in addition to the cancelation.
- The user triggers recalculation manually. As with the above case, a new function call is triggered in addition to the cancelation.

You *must* implement a cancellation handler for every streaming function. Asynchronous, non-streaming functions may or may not be cancelable; it's up to you. Synchronous functions cannot be canceled.

To make a function cancelable, set the option `"cancelable": true` in the `options` property for the custom function in the registration JSON file.

The following code shows the previous example with cancellation implemented. In the code, the `caller` object contains an `onCanceled` function must be defined for each cancelable custom function.

```js
function incrementValue(increment, caller){ 
    var result = 0;
    var timer = setInterval(function(){
         result += increment;
         caller.setResult(result);
    }, 1000);

    caller.onCanceled = function(){
        clearInterval(timer);
    }
}
```

## Saving and sharing state

Custom functions can save data in global JavaScript variables. In subsequent calls, your custom function may use the values saved in these variables. Saved state is useful when users add the same custom function to more than one cell, because all the instances of the function can share the state. For example, you may save the data returned from a call to a web resource to avoid making additional calls to the same web resource.

The following code shows an implementation of the previous temperature-streaming function that saves state globally. Note the following about this code:

- `refreshTemperature` is a streamed function that reads the temperature of a particular thermometer every second. New temperatures are saved in the `savedTemperatures` variable, but does not directly update the cell value. It should not be directly called from a worksheet cell, *so it is not registered in the JSON file*.
- `streamTemperature` updates the temperature values displayed in the cell every second and it uses `savedTemperatures` variable as its data source. It must be registered in the JSON file, and named with all upper-case letters, `STREAMTEMPERATURE`.
- Users may call `streamTemperature` from several cells in the Excel UI. Each call reads data from the same `savedTemperatures` variable.

```js
var savedTemperatures{};

function streamTemperature(thermometerID, caller){ 
     if(!savedTemperatures[thermometerID]){
         refreshTemperatures(thermometerID); // starts fetching temperatures if the thermometer hasn't been read yet
     }

     function getNextTemperature(){
         caller.setResult(savedTemperatures[thermometerID]); // setResult sends the saved temperature value to Excel.
         setTimeout(getNextTemperature, 1000); // Wait 1 second before updating Excel again.
     }
     getNextTemperature();
}

function refreshTemperature(thermometerID){
     sendWebRequest(thermometerID, function(data){
         savedTemperatures[thermometerID] = data.temperature;
     });
     setTimeout(function(){
         refreshTemperature(thermometerID);
     }, 1000); // Wait 1 second before reading the thermometer again, and then update the saved temperature of thermometerID.
}
```

## Working with ranges of data

Your custom function can take a range of data as a parameter, or you can return a range of data from a custom function.

For example, suppose that your function returns the second highest value from a range of numbers stored in Excel. The following function takes the parameter `values`, which is an `Excel.CustomFunctionDimensionality.matrix` parameter type. Note that in the registration JSON for this function, you would set the `type` child property, of the object in the `parameters` array, to `matrix`.

```js
function secondHighest(values){ 
     var highest = values[0][0], secondHighest = values[0][0];
     for(var i = 0; i < values.length; i++){
         for(var j = 1; j < values[i].length; j++){
             if(values[i][j] >= highest){
                 secondHighest = highest;
                 highest = values[i][j];
             }
             else if(values[i][j] >= secondHighest){
                 secondHighest = values[i][j];
             }
         }
     }
     return secondHighest;
 }
```

## Known issues

The following features aren't yet supported in the Developer Preview.

- Help URLs and parameter descriptions are not yet used by Excel.
- Custom functions are not available on Excel for mobile clients.
- Currently, add-ins rely on a hidden browser process to run asynchronous custom functions. In the future, JavaScript will run directly on some platforms to ensure custom functions are faster and use less memory. Additionally, the HTML page referenced by the `<Page>` element in the manifest won’t be needed for most platforms because Excel will run the JavaScript directly. To prepare for this change, ensure your custom functions do not use the web page DOM.
- Volatile functions (those which recalculate automatically whenever unrelated data changes in the spreadsheet) are not yet supported.

## Changelog

- **Nov 7, 2017**: Shipped the custom functions preview and samples
- **Nov 20, 2017**: Fixed compatibility bug for those using builds 8801 and later
- **Nov 28, 2017**: Shipped support for cancellation on asynchronous functions (requires change for streaming functions)
- **May 7, 2018**: Shipped support for Mac, Excel Online, and synchronous functions running in-process
