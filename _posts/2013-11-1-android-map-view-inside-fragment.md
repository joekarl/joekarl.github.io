---
layout: post
title: Using an Android MapView inside of a Fragment
---

One of the more annoying aspects of Android programming is that while basically anything is possible with the SDK, there are lots of straight forward things that aren't documented very well. 

I ran into one of those things today. Now there is all sorts of documentation that show how to use a MapFragment to display a MapView. Unfortunately if you are looking to just include a MapView inside of a Fragment, that documentation does you no good. You can't include a Fragment inside of a Fragment (as of right now) because the Fragment lifecycles get all screwed up when you go to create the Fragment view. 

<div class="well">Why not just use a MapFragment + a new Activity?
There are times when you just can't. My case was needing to show some arbitrary views above the MapView (so couldn't use a MapFragment) and still make everything work with a NavigationDrawer (which requires the subviews to all be fragments).</div>

###Solution
The solution I came to was to include the MapView directly in my Fragment layout and then implement the MapViews lifecycle calls.

Note the initializeMap() method. This is where you can setup the map with markers, set listeners, etc...

Also note calling initializeMap() in the onResume() fragment lifecycle method

Layout
    
    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >

        <TextView
            android:id="@+id/arbitraryView"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content" />

        <com.google.android.gms.maps.MapView
              android:id="@+id/map"
              android:layout_width="match_parent"
              android:layout_height="0dp"
              android:layout_weight="1"/>

    </LinearLayout>

Fragment

    public class CustomMapFragment extends Fragment {
        private GoogleMap googleMap;
        private MapView mapView;
        private boolean mapsSupported = true;

        @Override
        public void onActivityCreated(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            try {
                MapsInitializer.initialize(getActivity());
            } catch (GooglePlayServicesNotAvailableException e) {
                mapsSupported = false;
            }

            if (mapView != null) {
                mapView.onCreate(savedInstanceState);
            }
            initializeMap();
        }

        private void initializeMap() {
            if (googleMap == null && mapsSupported) {
                mapView = (MapView) getActivity().findViewById(R.id.map);
                googleMap = mapView.getMap();
                //setup markers etc...
            }
        }

        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
                Bundle savedInstanceState) {
            final LinearLayout parent = (LinearLayout) inflater.inflate(R.layout.nearby_layout, container, false);
            mapView = (MapView) parent.findViewById(R.id.map);
            return parent;
        }

        @Override
        public void onSaveInstanceState(Bundle outState) {
            super.onSaveInstanceState(outState);
            mapView.onSaveInstanceState(outState);
        }

        @Override
        public void onResume() {
            super.onResume();
            mapView.onResume();
            initializeMap();
        }

        @Override
        public void onPause() {
            super.onPause();
            mapView.onPause();
        }

        @Override
        public void onDestroy() {
            super.onDestroy();
            mapView.onDestroy();
        }

        @Override
        public void onLowMemory() {
            super.onLowMemory();
            mapView.onLowMemory();
        }
    }