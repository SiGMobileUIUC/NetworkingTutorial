# NetworkingTutorial
**API Calls, JSON, and Async tasks**
![alt tag](https://www-s.acm.illinois.edu/sigmobile/sig_mobile-01.png)

Hey! Today we're going to learn about API calls, parsing JSON, and creating an Async task. The API we're going to use is none other than our own CUMTD API, which returns JSONified data on bus activity. Lets create an app that lets you search for bus stops and return stop data. It will use CUMTD's autocomplete API to autocomplete stops from what you type. 

Since we're going to be making API calls on the internet, we want to define the permission in the app for Android to use internet. This can be done in your **AndroidManifest.xml**. Add the following line to your manifest outside the **< application />** tag:

```XML
<uses-permission android:name="android.permission.INTERNET" />
```

Next, you want to add a ListView and TextView (title) to your activity_main.xml, this view will be used to display the JSON data you'll receive.

![alt tag](http://i.imgur.com/51Id5Sq.png)

Instantiate those in your Java code.

```Java
EditText edtStopSearch;
ListView lstStops;

@Override
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	setContentView(R.layout.activity_main);

	edtStopSearch = (EditText) findViewById(R.id.editText);
	lstStops = (ListView) findViewById(R.id.listView);
```

First you want to add a TextchangeListener, so that when you type into the text bar, your app can grab it and run actions:

```Java
edtStopSearch.addTextChangedListener(new TextWatcher() {
```

There are three methods that we want to Override in the Listener class. Let's implement them here:

```Java
@Override
public void afterTextChanged(Editable s) {
	//nothing
}

@Override
public void beforeTextChanged(CharSequence s, int start, int count, int after) {
	//nothing
}

```

afterTextChanged and beforeTextChanges are not important to us. To implement an autocomplete for the CUMTD API, we want to override the onTextChanged method:

```Java
@Override
public void onTextChanged(CharSequence s, int start, int before, int count) {
	String userText = edtStopSearch.getText().toString();
	new MyAsyncTask().execute(userText);
}
```

The method will create something called a new Async Task. Async Tasks are meant to run network operations because Android prevents them from being run on the UI Thread (the main thread of Android). Async Tasks typically run network-related tasks that are a couple seconds long. Next, we'll define an Async Task class inline to define some key methods.

Create a private Async Task class that extends the AsyncTask class provided by the Android system:

```Java
private class MyAsyncTask extends AsyncTask<String, Void, String> {

protected void onPreExecute() {
	//nothing
}
```

There are two main methods we want to define in the class. a doInBackground method to run the API call in the background, and another to parse the received JSON into manageable text. 

First, we will create the doInBackground method that takes A String array (String...):

```Java
protected String doInBackground(String... strings) {
```

Define a **StringBuilder** to contain the JSON. next, you also want to create an **httpClient** object and an **HttpGet** object to first initialize the client, and then pull the http link. 

```Java
StringBuilder builder = new StringBuilder();
HttpClient client = new DefaultHttpClient();
HttpGet httpGet = new HttpGet(getString(R.string.autocompleteAPI) + strings[0]);
```
A couple of things to notice here:
**httpGet** is constructed from 2 string parts. The first part gets the string **autocompleteAPI**, which i will talk about below. This is the http link for the CUMTD API call provided on their website. The second thing is the part of the query that you are inputting, the bus stop. That is passed in as the first position of the String array, the 0th position (strings[0]).


Good Android practice is to not put links in code. Instead, were going to replace the http link with a string object, **autocompleteAPI** in the app/src/main/res/values/string.xml as so:

```XML
 <string name="autocompleteAPI">http://www.cumtd.com/autocomplete/Stops/v1.0/json/search?query=</string>
```
Next, were going to pull JSON from HTTP with the following chunk of code. This is a very generic request; the code can be copy+pasted regularly:

```Java
try {
	HttpResponse response = client.execute(httpGet);
	StatusLine statusLine = response.getStatusLine();
	int statusCode = statusLine.getStatusCode();
	if(statusCode == 200) {
		HttpEntity entity = response.getEntity();
		InputStream content = entity.getContent();
		BufferedReader reader = new BufferedReader(new InputStreamReader(content));
		String line;
		while((line = reader.readLine()) != null) {
			builder.append(line);
		}
	} 
	else {
		Log.e(MainActivity.class.toString(), "Failed to get JSON object");
	}
} 
catch(IOException e) {
	e.printStackTrace();
}
```

This code grabs the HTTP response and checks the status code. If the code is 200, the all clear code, then we can proceed with appending the **JSON** to the **StringBuilder** (builder) that we created. Otherwise, we'll output a failure message to the Log.
The last thing we need to do is return our **StringBuilder**:

```Java
return builder.toString();
```

The second method we're going to implement is the **onPostExecute** method. 
This method will parse the String containing our JSON for all the names of the stops, the "n" tags in each JSON object. We will add each stop to an ArrayList, and then output it:

```Java
protected void onPostExecute(String result) {
	query = result;

	try {
		JSONArray jsonArray = new JSONArray(query);

		for(int i = 0; i < jsonArray.length(); i++) {
			String name = jsonArray.getJSONObject(i).getString("n");
			results.add(name);
		}

		ArrayAdapter<String> arrayAdapter = new ArrayAdapter<String>(getApplicationContext(), android.R.layout.simple_list_item_1, results);
		lstStops.setAdapter(arrayAdapter);
	} 
	catch (Exception e) {
		e.printStackTrace();
	}
}
```

Lets try to run our app now! It should be able to search our stops!

 
![alt tag](http://i.imgur.com/E66d7yp.png)


**WHOOPS**. That text is white!

It turns out that Android is actually pretty stupid when it comes to things like this, and you have to manually make your text black. (It's a shame, iOS actually has some pretty good sensors to detect and change this automatically).

We can fix this by overriding a method of the **ArrayAdapter** class to change the text we're pushing out to be the color black. Modify your **ArrayAdapter< String >** instantiation to override a method inline:

```Java
ArrayAdapter<String> arrayAdapter = new ArrayAdapter<String>(getApplicationContext(), android.R.layout.simple_list_item_1, results) {
	@Override
	public View getView(int position, View convertView, ViewGroup parent) {
		View view = super.getView(position, convertView, parent);
		TextView textView = (TextView) view.findViewById(android.R.id.text1);
		textView.setTextColor(Color.BLACK);
		return view;
	}
};
```

Try running your code again, it should have black text now!

![alt tag](http://i.imgur.com/Ukn7zVU.png)

One bug that you may or may not have noticed, is that when you type something, your ListView updates with stops, but when you delete what you typed and then write something else, new stops get appended, while the old ones do not. This happens simply because we do not clear our ArrayAdapter every time we type something new. This can be done by adding a simple line to clear the Adapter in our **onPostExecute** method:

```Java
results.clear();
```

A last bug that you may have encountered is, when you press space, the app crashes. This is because were appending what we type to the end of the http link to get, but spaces dont register in URL's. What is actually done in links, is that spaces are represented by +'s. This can be implemented in our code by simply replacing spaces with +'s in our string before we make our **HttpGet** request in our **onTextChanged** method. Also, were going to set everything to lower case at this point for the sake of consistency:

```Java
@Override
public void onTextChanged(CharSequence s, int start, int before, int count) {
	String userText = edtStopSearch.getText().toString();
	userText = userText.replace(" ", "+").toLowerCase();
	new MyAsyncTask().execute(userText);
}
```

And thats it! You've created an app that can make API Calls and parse JSON on an AsyncTask thread:

![alt tag](http://i.imgur.com/CSQNzMs.png)


if you have any questions, send an email to *sigmobile-l@acm.uiuc.edu*!

![alt tag](https://www-s.acm.illinois.edu/sigmobile/sig_mobile-01.png)
