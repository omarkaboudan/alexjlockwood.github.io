---
layout: post
title: Life Before Loaders (part 1)
date: 2012-07-06
permalink: /2012/07/loaders-and-loadermanager-background.html
comments: true
---

<p>This post gives a brief introduction to <code>Loader</code>s and the <code>LoaderManager</code>. The first section describes how data was loaded prior to the release of Android 3.0, pointing out out some of the flaws of the pre-Honeycomb APIs. The second section defines the purpose of each class and summarizes their powerful ability in asynchronously loading data.</p>

<p>This is the first of a series of posts I will be writing on Loaders and the LoaderManager:</p>

<ul>
<li><b>Part 1:</b> <a href="http://www.androiddesignpatterns.com/2012/07/loaders-and-loadermanager-background.html">Life Before Loaders</a></li>
<li><b>Part 2:</b> <a href="http://www.androiddesignpatterns.com/2012/07/understanding-loadermanager.html">Understanding the LoaderManager</a></li>
<li><b>Part 3:</b> <a href="http://www.androiddesignpatterns.com/2012/08/implementing-loaders.html">Implementing Loaders</a></li>
<li><b>Part 4:</b> <a href="http://www.androiddesignpatterns.com/2012/09/tutorial-loader-loadermanager.html">Tutorial: AppListLoader</a></li>
</ul>

<p>If you know nothing about <code>Loader</code>s and the <code>LoaderManager</code>, I strongly recommend you read the <a href="http://developer.android.com/guide/components/loaders.html">documentation</a> before continuing forward.</p>

<h4>The Not-So-Distant Past</h4>

<p>Before Android 3.0, many Android applications lacked in responsiveness. UI interactions glitched, transitions between activities lagged, and ANR (Application Not Responding) dialogs rendered apps totally useless. This lack of responsiveness stemmed mostly from the fact that developers were performing queries on the UI thread--a very poor choice for lengthy operations like loading data.</p>

<p>While the <a href="http://developer.android.com/guide/practices/responsiveness.html">documentation</a> has always stressed the importance of instant feedback, the pre-Honeycomb APIs simply did not encourage this behavior. Before Loaders, cursors were primarily managed and queried for with two (now deprecated) <code>Activity</code> methods:</p>

<ul>
<li><p><code>public void startManagingCursor(Cursor)</code></p>

<p>Tells the activity to take care of managing the cursor's lifecycle based on the activity's lifecycle. The cursor will automatically be deactivated (<code>deactivate()</code>) when the activity is stopped, and will automatically be closed (<code>close()</code>) when the activity is destroyed. When the activity is stopped and then later restarted, the Cursor is re-queried (<code>requery()</code>) for the most up-to-date data.</p></li><!--more-->

<li><p><code>public Cursor managedQuery(Uri, String, String, String, String)</code></p>

<p>A wrapper around the <code>ContentResolver</code>'s <code>query()</code> method. In addition to performing the query, it begins management of the cursor (that is, <code>startManagingCursor(cursor)</code> is called before it is returned).</p></li>
</ul>

<p>While convenient, these methods were deeply flawed in that they performed queries on the UI thread. What's more, the "managed cursors" did not retain their data across <code>Activity</code> configuration changes. The need to <code>requery()</code> the cursor's data in these situations was unnecessary, inefficient, and made orientation changes clunky and sluggish as a result.</p>

<h4>The Problem with "Managed <code>Cursor</code>s"</h4>

<p>Let's illustrate the problem with "managed cursors" through a simple code sample. Given below is a <code>ListActivity</code> that loads data using the pre-Honeycomb APIs. The activity makes a query to the <code>ContentProvider</code> and begins management of the returned cursor. The results are then bound to a <code>SimpleCursorAdapter</code>, and are displayed on the screen in a <code>ListView</code>. The code has been condensed for simplicity.</p>

<p>
<pre class="brush:java">public class SampleListActivity extends ListActivity {

  private static final String[] PROJECTION = new String[] {"_id", "text_column"};

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Performs a "managed query" to the ContentProvider. The Activity 
    // will handle closing and requerying the cursor.
    //
    // WARNING!! This query (and any subsequent re-queries) will be
    // performed on the UI Thread!!
    Cursor cursor = managedQuery(
        CONTENT_URI,  // The Uri constant in your ContentProvider class
        PROJECTION,   // The columns to return for each data row
        null,         // No where clause
        null,         // No where clause
        null          // No sort order
        );

    String[] dataColumns = { "text_column" };
    int[] viewIDs = { R.id.text_view };
 
    // Create the backing adapter for the ListView.
    //
    // WARNING!! While not readily obvious, using this constructor will 
    // tell the CursorAdapter to register a ContentObserver that will
    // monitor the underlying data source. As part of the monitoring
    // process, the ContentObserver will call requery() on the cursor 
    // each time the data is updated. Since Cursor#requery() is performed 
    // on the UI thread, this constructor should be avoided at all costs!
    SimpleCursorAdapter adapter = new SimpleCursorAdapter(
        this,                // The Activity context
        R.layout.list_item,  // Points to the XML for a list item
        cursor,              // Cursor that contains the data to display
        dataColumns,         // Bind the data in column "text_column"...
        viewIDs              // ...to the TextView with id "R.id.text_view"
        );

    // Sets the ListView's adapter to be the cursor adapter that was 
    // just created.
    setListAdapter(adapter);
  }
}
</pre>
</p>

<p>There are three problems with the code above. If you have understood this post so far, the first two shouldn't be difficult to spot:</p>

<ol>

<li value="1"><p><code>managedQuery</code> performs a query on the main UI thread. This leads to unresponsive apps and should no longer be used.</p></li>

<li value="2"><p>As seen in the <code>Activity.java</code> <a href="http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/1.5_r4/android/app/Activity.java#Activity.managedQuery%28android.net.Uri%2Cjava.lang.String%5B%5D%2Cjava.lang.String%2Cjava.lang.String%29">source code</a>, the call to <code>managedQuery</code> begins management of the returned cursor with a call to <code>startManagingCursor(cursor)</code>. Having the activity manage the cursor seems convenient at first, as we no longer need to worry about deactivating/closing the cursor ourselves. However, this signals the activity to call <code>requery()</code> on the cursor <a href="http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/1.5_r4/android/app/Activity.java#3503">each time the activity returns from a stopped state</a>, and therefore puts the UI thread at risk. This cost significantly outweighs the convenience of having the activity deactivate/close the cursor for us.</p></li>

<li value="3"><p>The <code>SimpleCursorAdapter</code> constructor (line 33) is deprecated and should not be used. The problem with this constructor is that it will have the <code>SimpleCursorAdapter</code> auto-requery its data when changes are made. More specifically, the CursorAdapter will register a ContentObserver that monitors the underlying data source for changes, calling <code>requery()</code> on its bound cursor each time the data is modified. The <a href="http://developer.android.com/reference/android/widget/SimpleCursorAdapter.html#SimpleCursorAdapter(android.content.Context, int, android.database.Cursor, java.lang.String[], int[], int)">standard constructor</a> should be used instead (if you intend on loading the adapter's data with a <code>CursorLoader</code>, make sure you pass <code>0</code> as the last argument). Don't worry if you couldn't spot this one... it's a very subtle bug.</p></li>

</ol>

<p>With the first Android tablet about to be released, something had to be done to encourage UI-friendly development. The larger, 7-10" Honeycomb tablets called for more complicated, interactive, multi-paned layouts. Further, the introduction of the <code>Fragment</code> meant that applications were about to become more dynamic and event-driven. A simple, single-threaded approach to loading data could no longer be encouraged. Thus, the <code>Loader</code> and the <code>LoaderManager</code> were born.</p>

<h4>Android 3.0, Loaders, and the LoaderManager</h4>

<p>Prior to Honeycomb, it was difficult to manage cursors, synchronize correctly with the UI thread, and ensure all queries occured on a background thread. Android 3.0 introduced the <code>Loader</code> and <code>LoaderManager</code> classes to help simplify the process. Both classes are available for use in the Android Support Library, which supports all Android platforms back to Android 1.6.</p>

<p>The new <code>Loader</code> API is a huge step forward, and significantly improves the user experience. <code>Loader</code>s ensure that all cursor operations are done asynchronously, thus eliminating the possibility of blocking the UI thread. Further, when managed by the <code>LoaderManager</code>, <code>Loader</code>s retain their existing cursor data across the activity instance (for example, when it is restarted due to a configuration change), thus saving the cursor from unnecessary, potentially expensive re-queries. As an added bonus, <code>Loader</code>s are intelligent enough to monitor the underlying data source for updates, re-querying automatically when the data is changed.</p>

<h4>Conclusion</h4>

<p>Since the introduction of <code>Loader</code>s in Honeycomb and Compatibility Library, Android applications have changed for the better. Making use of the now deprecated <code>startManagingCursor</code>&nbsp;and <code>managedQuery</code>&nbsp;methods are extremely discouraged; not only do they slow down your app, but they can potentially bring it to a screeching halt. <code>Loader</code>s, on the other hand, significantly speed up the user experience by offloading the work to a separate background thread.</p>

<p>In the next post (titled <a href="http://www.androiddesignpatterns.com/2012/07/understanding-loadermanager.html">Understanding the LoaderManager</a>), we will go more in-depth on how to fix these problems by completing the transition from "managed cursors" to making use of <code>Loader</code>s and the <code>LoaderManager</code>.</p>

<p>Don't forget to +1 this blog in the top right corner if you found this helpful! :)</p>