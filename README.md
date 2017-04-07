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

We will need this code portion:

```       
            // These two need to be declared outside the try/catch
            // so that they can be closed in the finally block.
            HttpURLConnection urlConnection = null;
            BufferedReader reader = null;

            // Will contain the raw JSON response as a string.
            String forecastJsonStr = null;

            try {
                // Construct the URL for the OpenWeatherMap query
                // Possible parameters are avaiable at OWM's forecast API page, at
                // http://openweathermap.org/API#forecast
                //MODIFIED FOR CITY OF THESSALONIKI, GREECE
                URL url = new URL("http://api.openweathermap.org/data/2.5/forecast/daily?id=734077&mode=json&units=metric&cnt=7");

                // Create the request to OpenWeatherMap, and open the connection
                urlConnection = (HttpURLConnection) url.openConnection();
                urlConnection.setRequestMethod("GET");
                urlConnection.connect();

                // Read the input stream into a String
                InputStream inputStream = urlConnection.getInputStream();
                StringBuffer buffer = new StringBuffer();
                if (inputStream == null) {
                    // Nothing to do.
                    return null;
                }
                reader = new BufferedReader(new InputStreamReader(inputStream));

                String line;
                while ((line = reader.readLine()) != null) {
                    // Since it's JSON, adding a newline isn't necessary (it won't affect parsing)
                    // But it does make debugging a *lot* easier if you print out the completed
                    // buffer for debugging.
                    buffer.append(line + "\n");
                }

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
        }
    }
}

```
paste the code inside method ```onCreateView()``` of class ```PlaceholderFragment```, just after the statement:

```
listView.setAdapter(...)
```

Executing the App at this time will result in Error and App Crash!

The error is of type ```NetworkOnMainThreadException``` and actually complains because we created a network connection inside a method of the Main Thread.

Parenthesis 2: Thread
![Threads](https://github.com/UomMobileDevelopment/Lesson03-material/blob/master/threads.png)

So, we will a class that will ease the creation of hreads and UI thread synchronization 

## [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask.html) ##

After studing the ```AsyncTask```, we will use it to transfer there the code that was offhandedly pasted inside ```PlaceholderFragment```.

We have to do some Refactoring:

   1. Rename PlaceholderFragment -> ForecastFragment
   2. Move ForecastFragment to new file
   3. Create a new AsyncTask child called FetchWeatherTask with the networking code snippet from above. This class should be created inside ForecastFragment, like this
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
END OF LESSON 2 - UNTIL HERE
--------------------------------

##JSON Parsing##

Quiz: Parse Out The Max Temp
https://classroom.udacity.com/courses/ud853/lessons/1469948762/concepts/16307786410923#


```
package com.example.android.sunshine.app;

import android.net.Uri;
import android.os.AsyncTask;
import android.os.Bundle;
import android.support.v4.app.Fragment;
import android.text.format.Time;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.Menu;
import android.view.MenuInflater;
import android.view.MenuItem;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ArrayAdapter;
import android.widget.ListView;

import org.json.JSONArray;
import org.json.JSONException;
import org.json.JSONObject;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.URL;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class ForecastFragment extends Fragment {

    ArrayAdapter<String> mForecastAdapter;

    public ForecastFragment() {
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setHasOptionsMenu(true);
    }

    @Override
    public void onCreateOptionsMenu(Menu menu, MenuInflater inflater) {
        inflater.inflate(R.menu.forecastfragment, menu);
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        int id = item.getItemId();

        if(id == R.id.action_refresh){
            FetchWeatherTask weatherTask = new FetchWeatherTask();
            weatherTask.execute("734077");
            return true;
        }
        return super.onOptionsItemSelected(item);
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {

        // Create some dummy data for the ListView.  Here's a sample weekly forecast
        String[] data = {
                "Mon 6/23 - Sunny - 31/17",
                "Tue 6/24 - Foggy - 21/8",
                "Wed 6/25 - Cloudy - 22/17",
                "Thurs 6/26 - Rainy - 18/11",
                "Fri 6/27 - Foggy - 21/10",
                "Sat 6/28 - TRAPPED IN WEATHERSTATION - 23/18",
                "Sun 6/29 - Sunny - 20/7"
        };
        List<String> weekForecast = new ArrayList<String>(Arrays.asList(data));


        // Now that we have some dummy forecast data, create an ArrayAdapter.
        // The ArrayAdapter will take data from a source (like our dummy forecast) and
        // use it to populate the ListView it's attached to.
        mForecastAdapter =
            new ArrayAdapter<String>(
                getActivity(), // The current context (this activity)
                R.layout.list_item_forecast, // The name of the layout ID.
                R.id.list_item_forecast_textview, // The ID of the textview to populate.
                weekForecast);

        View rootView = inflater.inflate(R.layout.fragment_main, container, false);

        // Get a reference to the ListView, and attach this adapter to it.
        ListView listView = (ListView) rootView.findViewById(R.id.listview_forecast);
        listView.setAdapter(mForecastAdapter);


        return rootView;
    }

    public class FetchWeatherTask extends AsyncTask<String,Void,String[]> {

        private final String LOG_TAG = FetchWeatherTask.class.getSimpleName();

        /* The date/time conversion code is going to be moved outside the asynctask later,
         * so for convenience we're breaking it out into its own method now.
         */
        private String getReadableDateString(long time){
            // Because the API returns a unix timestamp (measured in seconds),
            // it must be converted to milliseconds in order to be converted to valid date.
            SimpleDateFormat shortenedDateFormat = new SimpleDateFormat("EEE MMM dd");
            return shortenedDateFormat.format(time);
        }

        /**
         * Prepare the weather high/lows for presentation.
         */
        private String formatHighLows(double high, double low) {
            // For presentation, assume the user doesn't care about tenths of a degree.
            long roundedHigh = Math.round(high);
            long roundedLow = Math.round(low);

            String highLowStr = roundedHigh + "/" + roundedLow;
            return highLowStr;
        }

        /**
         * Take the String representing the complete forecast in JSON Format and
         * pull out the data we need to construct the Strings needed for the wireframes.
         *
         * Fortunately parsing is easy:  constructor takes the JSON string and converts it
         * into an Object hierarchy for us.
         */
        private String[] getWeatherDataFromJson(String forecastJsonStr, int numDays)
                throws JSONException {

            // These are the names of the JSON objects that need to be extracted.
            final String OWM_LIST = "list";
            final String OWM_WEATHER = "weather";
            final String OWM_TEMPERATURE = "temp";
            final String OWM_MAX = "max";
            final String OWM_MIN = "min";
            final String OWM_DESCRIPTION = "main";

            JSONObject forecastJson = new JSONObject(forecastJsonStr);
            JSONArray weatherArray = forecastJson.getJSONArray(OWM_LIST);

            // OWM returns daily forecasts based upon the local time of the city that is being
            // asked for, which means that we need to know the GMT offset to translate this data
            // properly.

            // Since this data is also sent in-order and the first day is always the
            // current day, we're going to take advantage of that to get a nice
            // normalized UTC date for all of our weather.

            Time dayTime = new Time();
            dayTime.setToNow();

            // we start at the day returned by local time. Otherwise this is a mess.
            int julianStartDay = Time.getJulianDay(System.currentTimeMillis(), dayTime.gmtoff);

            // now we work exclusively in UTC
            dayTime = new Time();

            String[] resultStrs = new String[numDays];
            for(int i = 0; i < weatherArray.length(); i++) {
                // For now, using the format "Day, description, hi/low"
                String day;
                String description;
                String highAndLow;

                // Get the JSON object representing the day
                JSONObject dayForecast = weatherArray.getJSONObject(i);

                // The date/time is returned as a long.  We need to convert that
                // into something human-readable, since most people won't read "1400356800" as
                // "this saturday".
                long dateTime;
                // Cheating to convert this to UTC time, which is what we want anyhow
                dateTime = dayTime.setJulianDay(julianStartDay+i);
                day = getReadableDateString(dateTime);

                // description is in a child array called "weather", which is 1 element long.
                JSONObject weatherObject = dayForecast.getJSONArray(OWM_WEATHER).getJSONObject(0);
                description = weatherObject.getString(OWM_DESCRIPTION);

                // Temperatures are in a child object called "temp".  Try not to name variables
                // "temp" when working with temperature.  It confuses everybody.
                JSONObject temperatureObject = dayForecast.getJSONObject(OWM_TEMPERATURE);
                double high = temperatureObject.getDouble(OWM_MAX);
                double low = temperatureObject.getDouble(OWM_MIN);

                highAndLow = formatHighLows(high, low);
                resultStrs[i] = day + " - " + description + " - " + highAndLow;
            }

            for (String s : resultStrs) {
                Log.v(LOG_TAG, "Forecast entry: " + s);
            }
            return resultStrs;

        }

        @Override
        protected String[] doInBackground(String... params) {

            // If there's no city code, there's nothing to look up.  Verify size of params.
            if (params.length == 0) {
                return null;
            }

            // These two need to be declared outside the try/catch
            // so that they can be closed in the finally block.
            HttpURLConnection urlConnection = null;
            BufferedReader reader = null;

                // Will contain the raw JSON response as a string.
                String forecastJsonStr = null;
                String weatherFormat = "json";
                int numDays = 7;
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
                            .appendQueryParameter(daysParam, Integer.toString(numDays))
                            .appendQueryParameter(apiKeyParam, BuildConfig.OPEN_WEATHER_MAP_API_KEY)
                            .build();

                    URL url = new URL(builtUri.toString());

                    Log.v(LOG_TAG, "Built URI: "+builtUri.toString());

                // Create the request to OpenWeatherMap, and open the connection
                urlConnection = (HttpURLConnection) url.openConnection();
                urlConnection.setRequestMethod("GET");
                urlConnection.connect();

                // Read the input stream into a String
                InputStream inputStream = urlConnection.getInputStream();
                StringBuffer buffer = new StringBuffer();
                if (inputStream == null) {
                    // Nothing to do.
                    return null;
                }
                reader = new BufferedReader(new InputStreamReader(inputStream));

                String line;
                while ((line = reader.readLine()) != null) {
                    // Since it's JSON, adding a newline isn't necessary (it won't affect parsing)
                    // But it does make debugging a *lot* easier if you print out the completed
                    // buffer for debugging.
                    buffer.append(line + "\n");
                }

                if (buffer.length() == 0) {
                    // Stream was empty.  No point in parsing.
                    return null;
                }
                forecastJsonStr = buffer.toString();
                Log.v(LOG_TAG,"Forecast JSON String: "+forecastJsonStr);
            } catch (IOException e) {
                Log.e(LOG_TAG, "Error ", e);
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
                        Log.e(LOG_TAG, "Error closing stream", e);
                    }
                }
            }
            try {
                return getWeatherDataFromJson(forecastJsonStr, numDays);
            } catch (JSONException e) {
                Log.e(LOG_TAG, e.getMessage(), e);
                e.printStackTrace();
            }

            // This will only happen if there was an error getting or parsing the forecast.
            return null;
        }

        @Override
        protected void onPostExecute(String[] result) {
            if (result != null) {
                mForecastAdapter.clear();
                for(String dayForecastStr : result) {
                    mForecastAdapter.add(dayForecastStr);
                }
                // New data is back from the server.  Hooray!
            }
        }
    }

}


```

## Second Screen (Activity)

We will now add a second screen - activity which will present the weather data of a specific day that was tapped on the forecast list view. 

We will begin with a simple [Toast (like a popup message box)](https://developer.android.com/guide/topics/ui/notifiers/toasts.html) to test the tapping function. We'll going to need an ```Item Click Listener```.

As you will see, we create the listener anonymously for code simplicity. So, with this temporary ```toaster``` solution, the correct function of the listener will be tested.


```
 listView.setOnItemClickListener(new AdapterView.OnItemClickListener(){
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                final String item = mForecastAdapter.getItem(position);
                Intent intent = new Intent(getActivity(), DetailActivity.class);
                intent.putExtra(Intent.EXTRA_TEXT, item);
                startActivity(intent);

            }
        });

```

Now that we are sure that tapping event actually works, lets move to the creation of the second Activity. 
Create a new class ```DetailActivity```



```
package com.example.android.sunshine.app;

import android.content.Intent;
import android.os.Bundle;
import android.support.v4.app.Fragment;
import android.support.v7.app.ActionBarActivity;
import android.view.LayoutInflater;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

public class DetailActivity extends ActionBarActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_detail);
        if (savedInstanceState == null) {
            getSupportFragmentManager().beginTransaction()
                    .add(R.id.container, new PlaceholderFragment())
                    .commit();
        }
    }


    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.detail, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        // Handle action bar item clicks here. The action bar will
        // automatically handle clicks on the Home/Up button, so long
        // as you specify a parent activity in AndroidManifest.xml.
        int id = item.getItemId();

        //noinspection SimplifiableIfStatement
        if (id == R.id.action_settings) {
            Intent intent = new Intent(this, SettingsActivity.class);
            startActivity(intent);
            return true;
        }

        return super.onOptionsItemSelected(item);
    }

    /**
     * A placeholder fragment containing a simple view.
     */
    public static class PlaceholderFragment extends Fragment {

        public PlaceholderFragment() {
        }

        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
                                 Bundle savedInstanceState) {

            Intent intent = getActivity().getIntent();
            View rootView = inflater.inflate(R.layout.fragment_detail, container, false);

            if(intent != null && intent.hasExtra(Intent.EXTRA_TEXT)){
                String text = intent.getStringExtra(Intent.EXTRA_TEXT);
                TextView detailText = (TextView)rootView.findViewById(R.id.detail_text);
                detailText.setText(text);
            }
            return rootView;
        }
    }
}
```

And the fragment_detail.xml

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/activity_detail"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    tools:context="com.example.android.sunshine.app.DetailActivity">

<TextView
    android:id="@+id/detail_text"
    android:text=""
    android:layout_width="wrap_content"
    android:layout_height="wrap_content" />

</RelativeLayout>

```


