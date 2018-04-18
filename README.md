# IHU Lesson02-material
### The second lesson includes the invocation of remote functions (Web Service) to retrieve weather forecasts from real weather services 

To begin, we should introduce and explain the Web Service notion.

We will be using the Open Weather map API http://openweathermap.org/ for weather forecast retrieval regarding our area.
Number one action is to create an account at openweathermap.org and get an API key. We will use this key to "sign" our API requests.

Example use, to retrieve the Thessaloniki forecast for the next 7 days:

```
api.openweathermap.org/data/2.5/forecast/daily?id=734077&units=metric&cnt=7&APPID=27949ea6b6dffa1dad1deb925c9b024b
```

Wait, what is this macaroni?

What is the HTTP Request after all?  Check here: http://www.tutorialspoint.com/http/http_requests

How can I use this in my java code?

Parenthesis: Java Logging

![logging types](https://github.com/UomMobileDevelopment/Lesson03-material/blob/master/logging-types.png)

We will need to write some code!

```       
            // These two need to be declared outside the try/catch
            // so that they can be closed in the finally block.
            HttpURLConnection urlConnection = null;
            BufferedReader reader = null;

            // Will contain the raw JSON response as a string.
            String forecastJsonStr = null;
```

declare the variables for the url connection and the buffered reader

```
            try {
                // Construct the URL for the OpenWeatherMap query
                // Possible parameters are avaiable at OWM's forecast API page, at
                // http://openweathermap.org/API#forecast
                //MODIFIED FOR CITY OF THESSALONIKI, GREECE
                URL url = new URL("http://api.openweathermap.org/data/2.5/forecast/daily?id=734077&mode=json&units=metric&cnt=7");
```

Create the URL. Note that this is not the optimum way!


```
                // Create the request to OpenWeatherMap, and open the connection
                urlConnection = (HttpURLConnection) url.openConnection();
                urlConnection.setRequestMethod("GET");
                urlConnection.connect();
```

create the url connection by using the method, ````url.openConnection()```
the connect() method sends the request.


```
                // Read the input stream into a String
                InputStream inputStream = urlConnection.getInputStream();
                StringBuffer buffer = new StringBuffer();
                if (inputStream == null) {
                    // Nothing to do.
                    return null;
                }
                reader = new BufferedReader(new InputStreamReader(inputStream));
````
Now it is time to read the response

```
                String line;
                while ((line = reader.readLine()) != null) {
                    // Since it's JSON, adding a newline isn't necessary (it won't affect parsing)
                    // But it does make debugging a *lot* easier if you print out the completed
                    // buffer for debugging.
                    buffer.append(line + "\n");
                }
```
Read the buffer line by line...

and handle possible exceptions...

```
                if (buffer.length() == 0) {
                    // Stream was empty.  No point in parsing.
                    return null;
                }
                forecastJsonStr = buffer.toString();
            } catch (IOException e) {
                Log.e("PlaceholderFragment", "Error ", e);
                // If the code didn't successfully get the weather data, there's no point in attemping
                // to parse it.
                return null;
            } finally{
                if (urlConnection != null) {
                    urlConnection.disconnect();
                }
                if (reader != null) {
                    try {
                        reader.close();
                    } catch (final IOException e) {
                        Log.e("PlaceholderFragment", "Error closing stream", e);
                    }
                }
            }

            return rootView;


```
paste the code inside method ```onCreateView()``` of class ```ForecastFragment```, just after the statement:

```
listView.setAdapter(...)
```

Executing the App at this time will result in Error and App Crash!

The error is of type ```NetworkOnMainThreadException``` and actually complains because we created a network connection inside a method of the Main Thread.

Parenthesis 2: Thread
![Threads](https://github.com/UomMobileDevelopment/Lesson03-material/blob/master/threads.png)

So, we will a class that will ease the creation of hreads and UI thread synchronization 

## [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask.html) ##

After studing the ```AsyncTask```, we will use it to transfer there the code that was offhandedly pasted inside ```ForecastFragment```.

We have to do some Refactoring:

   1. Create a new AsyncTask child called FetchWeatherTask with the networking code snippet from above. This class should be created inside ForecastFragment, like this
   ```
   public class FetchWeatherTask extends AsyncTask<Void,Void,Void> {
   ....
   }
   ```
   Don't forget to implement the abstract method ``` protected Void doInBackground(Void... params) {} ```

Now the App runs without crashing but our data is still dummy

We will add a button in the main menu that will trigger the weather fetching from the remote weather service. However we need to understand that this is only a debugging-oriented solution and should not be implemented in production systems. It is a BAD idea to depend on user interaction to load your data. A good App is like a good buttler, it should know when to serve neccessary info, the time you need it!  Relative video: https://www.youtube.com/watch?v=VFdIy0GjUEs

Furthermore, the data fetching task needs some improvements, cause it is rather binded to the UI Activity and therefore, any change to the UI (eg, a screen orientation change) will also interrupt data retrieval!!. But for now, lets stick to this solution.

Let's add the refresh button. It will be placed in the main menu, so we have to say few words about the Android MENU:

![Menu structure](https://github.com/UomMobileDevelopment/Lesson03-material/blob/master/menu.png)

We should first add a new xml file inside res/menu. 
Create the file ```forecastFragment.xml``` with contents:

```
<?xml version="1.0" encoding="utf-8"?>

<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <item
        android:id="@+id/action_refresh"
        android:title="@string/action_refresh"
        app:showAsAction="never">

    </item>

</menu>
```

Implement the following methods in ForecastFragment

```
@Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setHasOptionsMenu(true);
    }
```
This states that the Fragment has its own menu option (the Refresh option). Each fragment can have different menu options.


```
    @Override
    public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {
        inflater.inflate(R.menu.forecastfragment, menu);
    }
```
this code inflates (loads the option) 'Refresh' in the menu 


```
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        int id = item.getItemId();

       if(id == R.id.action_refresh){
            AsyncTask<Void, Void, Void> weatherTask = new FetchWeatherTask().execute();
            return true;
        }
        return super.onOptionsItemSelected(item);
    }
```
this snippet defines the code that will run when the 'Refresh' option is tapped. We add the creation and invocation of our new AsyncTask

# BUT! 
If we run the App and hit 'Refresh' we will get a nice CRASH!! Search the logs to find out why:

```
 Caused by: java.lang.SecurityException: Permission denied (missing INTERNET permission?)
```

We need to ask the system for the internet permission, in order for our app to be able to communicate with the outer world! We should add the following statement 

```  
<uses-permission android:name="android.permission.INTERNET" />
```

in the ```AndroidManifest.xml```, in this area:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.android.sunshine.app" >
   <uses-permission android:name="android.permission.INTERNET" />
....
```

Now the App should work smoothly! To validate the correct weather data retrieval, we can add a LOG statement in FetchWeatherTask to display the forecastJsonStr variable:

```
 forecastJsonStr = buffer.toString();
 Log.v(LOG_TAG,"Forecast JSON String: "+forecastJsonStr);
```

we can see the LOG output in the AndroidMonitor tab of Android Studio. Weather data should be fine by now.


We should now improve the functionality of the Web Service URL and introduce some parameterization in Fetch Weathet Task so it can take a city code as a paraemeter and fetch the weather for this specific city
```
 AsyncTask<String, Void, Void> weatherTask = new FetchWeatherTask().execute("734077");
```

See the file [greek-city-codes.csv](https://github.com/UomMobileDevelopment/Lesson03-material/blob/master/greek-city-codes.csv) for the codes for all major Greek cities

URL can be built parametrically with the help of the URI Builder:

```
                ......
                // Will contain the raw JSON response as a string.
                String forecastJsonStr = null;
                String weatherFormat = "json";
                String daysForecast = "7";
                String units = "metric";
    
                try {
                    // Construct the URL for the OpenWeatherMap query
                    // Possible parameters are avaiable at OWM's forecast API page, at
                    // http://openweathermap.org/API#forecast
                    //MODIFIED FOR CITY OF THESSALONIKI, GREECE
                    final String baseUrl = "http://api.openweathermap.org/data/2.5/forecast/daily?";
                            //"id=734077&mode=json&units=metric&cnt=7";
                    final String queryParam = "id";
                    final String formatParam= "mode";
                    final String unitsParam= "units";
                    final String daysParam = "cnt";
                    final String apiKeyParam = "APPID";
    
                    Uri builtUri = Uri.parse(baseUrl).buildUpon()
                            .appendQueryParameter(queryParam,params[0])
                            .appendQueryParameter(formatParam,weatherFormat)
                            .appendQueryParameter(unitsParam, units)
                            .appendQueryParameter(daysParam, daysForecast)
                            .appendQueryParameter(apiKeyParam, BuildConfig.OPEN_WEATHER_MAP_API_KEY)
                            .build();
    
                    URL url = new URL(builtUri.toString());
    
                    Log.v(LOG_TAG, "Built URI: "+builtUri.toString());

                // Create the request to OpenWeatherMap, and open the connection
                urlConnection = (HttpURLConnection) url.openConnection();  
                .......
```

---------------------------------
END OF LESSON 2
