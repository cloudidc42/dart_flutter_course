# Part 37: Maps & Location
## ขั้นตอนที่ 1361-1400

---

## 🎯 เป้าหมายของ Part นี้

- Google Maps integration
- Current location
- Markers, Polylines, Polygons
- Geocoding
- Location permissions

---

## ขั้นตอนที่ 1361: Setup

```yaml
# pubspec.yaml
dependencies:
  google_maps_flutter: ^2.7.0
  geolocator: ^12.0.0
  geocoding: ^3.0.0
  permission_handler: ^11.3.0
  flutter_polyline_points: ^2.0.0
```

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest>
  <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
  
  <application>
    <meta-data
      android:name="com.google.android.geo.API_KEY"
      android:value="YOUR_GOOGLE_MAPS_API_KEY"/>
  </application>
</manifest>
```

```xml
<!-- ios/Runner/AppDelegate.swift -->
<!-- หรือ Info.plist -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณเพื่อแสดงสถานที่ใกล้เคียง</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>แอปต้องการตำแหน่งของคุณ</string>
```

---

## ขั้นตอนที่ 1362: Location Service

```dart
import 'package:geolocator/geolocator.dart';
import 'package:permission_handler/permission_handler.dart';

class LocationService {
  static final LocationService _instance = LocationService._();
  factory LocationService() => _instance;
  LocationService._();

  // ขอ permission
  Future<bool> requestPermission() async {
    PermissionStatus status = await Permission.location.request();
    return status.isGranted;
  }

  // ตรวจสอบ permission
  Future<LocationPermission> checkPermission() async {
    return Geolocator.checkPermission();
  }

  // ตำแหน่งปัจจุบัน
  Future<Position?> getCurrentPosition() async {
    bool serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) return null;

    LocationPermission permission = await checkPermission();
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied) return null;
    }
    if (permission == LocationPermission.deniedForever) return null;

    return Geolocator.getCurrentPosition(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.high,
        distanceFilter: 10, // update เมื่อเคลื่อนที่ 10 เมตร
      ),
    );
  }

  // Stream ตำแหน่งแบบ real-time
  Stream<Position> getPositionStream() {
    return Geolocator.getPositionStream(
      locationSettings: const LocationSettings(
        accuracy: LocationAccuracy.high,
        distanceFilter: 10,
      ),
    );
  }

  // คำนวณระยะทาง
  double distanceBetween(
    double startLat, double startLng,
    double endLat, double endLng,
  ) {
    return Geolocator.distanceBetween(startLat, startLng, endLat, endLng);
  }

  // คำนวณทิศทาง
  double bearingBetween(
    double startLat, double startLng,
    double endLat, double endLng,
  ) {
    return Geolocator.bearingBetween(startLat, startLng, endLat, endLng);
  }
}
```

---

## ขั้นตอนที่ 1363: Google Maps Widget

```dart
import 'dart:async';
import 'package:flutter/material.dart';
import 'package:google_maps_flutter/google_maps_flutter.dart';
import 'package:geolocator/geolocator.dart';

class MapsPage extends StatefulWidget {
  const MapsPage({super.key});

  @override
  State<MapsPage> createState() => _MapsPageState();
}

class _MapsPageState extends State<MapsPage> {
  final Completer<GoogleMapController> _mapController = Completer();
  Set<Marker> _markers = {};
  Set<Polyline> _polylines = {};
  Set<Polygon> _polygons = {};
  Set<Circle> _circles = {};

  static const LatLng _bangkokCenter = LatLng(13.7563, 100.5018);

  // Bangkok landmarks
  static const List<Map<String, dynamic>> _landmarks = [
    {'name': 'Grand Palace', 'lat': 13.7500, 'lng': 100.4913, 'icon': '🏛️'},
    {'name': 'Wat Arun', 'lat': 13.7439, 'lng': 100.4888, 'icon': '⛩️'},
    {'name': 'Chatuchak Market', 'lat': 13.7999, 'lng': 100.5499, 'icon': '🛍️'},
    {'name': 'Suvarnabhumi Airport', 'lat': 13.6811, 'lng': 100.7477, 'icon': '✈️'},
  ];

  @override
  void initState() {
    super.initState();
    _loadMarkers();
  }

  void _loadMarkers() {
    Set<Marker> markers = _landmarks.map((landmark) {
      return Marker(
        markerId: MarkerId(landmark['name']),
        position: LatLng(landmark['lat'], landmark['lng']),
        infoWindow: InfoWindow(
          title: landmark['name'],
          snippet: landmark['icon'],
          onTap: () => _onMarkerTap(landmark['name']),
        ),
        icon: BitmapDescriptor.defaultMarkerWithHue(
          _landmarks.indexOf(landmark) * 60.0,
        ),
      );
    }).toSet();

    setState(() => _markers = markers);
  }

  void _onMarkerTap(String name) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Tapped: $name')),
    );
  }

  void _addCustomMarker(LatLng position) {
    String id = 'custom_${DateTime.now().millisecondsSinceEpoch}';
    setState(() {
      _markers.add(Marker(
        markerId: MarkerId(id),
        position: position,
        infoWindow: const InfoWindow(title: 'Custom Marker'),
        icon: BitmapDescriptor.defaultMarkerWithHue(BitmapDescriptor.hueBlue),
      ));
    });
  }

  Future<void> _goToCurrentLocation() async {
    Position? position = await LocationService().getCurrentPosition();
    if (position == null) return;

    GoogleMapController controller = await _mapController.future;
    controller.animateCamera(
      CameraUpdate.newCameraPosition(
        CameraPosition(
          target: LatLng(position.latitude, position.longitude),
          zoom: 16,
        ),
      ),
    );

    setState(() {
      _markers.add(Marker(
        markerId: const MarkerId('my_location'),
        position: LatLng(position.latitude, position.longitude),
        infoWindow: const InfoWindow(title: 'ตำแหน่งของฉัน'),
        icon: BitmapDescriptor.defaultMarkerWithHue(BitmapDescriptor.hueCyan),
      ));
    });
  }

  void _drawRoute() {
    // วาดเส้นทาง
    setState(() {
      _polylines.add(const Polyline(
        polylineId: PolylineId('route'),
        color: Colors.blue,
        width: 5,
        points: [
          LatLng(13.7500, 100.4913), // Grand Palace
          LatLng(13.7439, 100.4888), // Wat Arun
          LatLng(13.7563, 100.5018), // Bangkok center
        ],
      ));
    });
  }

  void _drawRadius() {
    setState(() {
      _circles.add(const Circle(
        circleId: CircleId('radius'),
        center: _bangkokCenter,
        radius: 2000, // 2km
        fillColor: Color(0x44FF0000),
        strokeColor: Colors.red,
        strokeWidth: 2,
      ));
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Bangkok Map')),
      body: Stack(
        children: [
          GoogleMap(
            initialCameraPosition: const CameraPosition(
              target: _bangkokCenter,
              zoom: 12,
            ),
            onMapCreated: (controller) => _mapController.complete(controller),
            markers: _markers,
            polylines: _polylines,
            polygons: _polygons,
            circles: _circles,
            myLocationEnabled: true,
            myLocationButtonEnabled: false,
            mapToolbarEnabled: false,
            zoomControlsEnabled: false,
            onLongPress: _addCustomMarker,
            // Map type
            mapType: MapType.normal,
          ),
          // Controls
          Positioned(
            right: 16,
            bottom: 100,
            child: Column(
              children: [
                FloatingActionButton(
                  heroTag: 'location',
                  onPressed: _goToCurrentLocation,
                  child: const Icon(Icons.my_location),
                ),
                const SizedBox(height: 8),
                FloatingActionButton(
                  heroTag: 'route',
                  onPressed: _drawRoute,
                  child: const Icon(Icons.directions),
                ),
                const SizedBox(height: 8),
                FloatingActionButton(
                  heroTag: 'radius',
                  onPressed: _drawRadius,
                  child: const Icon(Icons.radio_button_checked),
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1364: Geocoding

```dart
import 'package:geocoding/geocoding.dart';
import 'package:flutter/material.dart';

class GeocodingService {
  // Address -> LatLng
  Future<Location?> getCoordinates(String address) async {
    try {
      List<Location> locations = await locationFromAddress(address);
      return locations.isNotEmpty ? locations.first : null;
    } catch (e) {
      print('Geocoding error: $e');
      return null;
    }
  }

  // LatLng -> Address
  Future<Placemark?> getAddress(double lat, double lng) async {
    try {
      List<Placemark> placemarks = await placemarkFromCoordinates(lat, lng);
      return placemarks.isNotEmpty ? placemarks.first : null;
    } catch (e) {
      print('Reverse geocoding error: $e');
      return null;
    }
  }

  String formatAddress(Placemark place) {
    return [
      place.street,
      place.subLocality,
      place.locality,
      place.administrativeArea,
      place.country,
    ].where((s) => s != null && s.isNotEmpty).join(', ');
  }
}

// ─── Search Location Widget ───
class LocationSearchWidget extends StatefulWidget {
  final Function(double lat, double lng) onLocationSelected;
  const LocationSearchWidget({super.key, required this.onLocationSelected});

  @override
  State<LocationSearchWidget> createState() => _LocationSearchWidgetState();
}

class _LocationSearchWidgetState extends State<LocationSearchWidget> {
  final TextEditingController _controller = TextEditingController();
  final GeocodingService _service = GeocodingService();
  bool _isLoading = false;
  String? _result;

  Future<void> _search() async {
    String query = _controller.text.trim();
    if (query.isEmpty) return;

    setState(() => _isLoading = true);

    Location? location = await _service.getCoordinates(query);

    setState(() {
      _isLoading = false;
      if (location != null) {
        _result = 'Lat: ${location.latitude.toStringAsFixed(4)}, '
            'Lng: ${location.longitude.toStringAsFixed(4)}';
        widget.onLocationSelected(location.latitude, location.longitude);
      } else {
        _result = 'ไม่พบตำแหน่งที่ค้นหา';
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            TextField(
              controller: _controller,
              decoration: InputDecoration(
                hintText: 'ค้นหาสถานที่...',
                prefixIcon: const Icon(Icons.search),
                suffixIcon: _isLoading
                    ? const SizedBox(
                        width: 20,
                        height: 20,
                        child: CircularProgressIndicator(strokeWidth: 2),
                      )
                    : IconButton(
                        icon: const Icon(Icons.send),
                        onPressed: _search,
                      ),
                border: const OutlineInputBorder(),
              ),
              onSubmitted: (_) => _search(),
            ),
            if (_result != null) ...[
              const SizedBox(height: 8),
              Text(_result!),
            ],
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 1365: Nearby Places

```dart
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:geolocator/geolocator.dart';

class Place {
  final String id;
  final String name;
  final String category;
  final double lat;
  final double lng;
  final double rating;

  const Place({
    required this.id,
    required this.name,
    required this.category,
    required this.lat,
    required this.lng,
    required this.rating,
  });

  double distanceTo(double lat, double lng) {
    return Geolocator.distanceBetween(this.lat, this.lng, lat, lng);
  }
}

class NearbyPlacesScreen extends StatefulWidget {
  const NearbyPlacesScreen({super.key});

  @override
  State<NearbyPlacesScreen> createState() => _NearbyPlacesScreenState();
}

class _NearbyPlacesScreenState extends State<NearbyPlacesScreen> {
  Position? _currentPosition;
  bool _isLoading = true;
  String _selectedCategory = 'All';

  // Mock places data
  static final List<Place> _allPlaces = [
    const Place(id: '1', name: 'Restaurant A', category: 'Food', lat: 13.7560, lng: 100.5010, rating: 4.5),
    const Place(id: '2', name: 'Coffee Shop B', category: 'Cafe', lat: 13.7570, lng: 100.5020, rating: 4.2),
    const Place(id: '3', name: 'Hospital C', category: 'Health', lat: 13.7550, lng: 100.5000, rating: 4.8),
    const Place(id: '4', name: 'School D', category: 'Education', lat: 13.7580, lng: 100.5030, rating: 4.0),
    const Place(id: '5', name: 'Mall E', category: 'Shopping', lat: 13.7540, lng: 100.4990, rating: 4.3),
  ];

  static const List<String> _categories = ['All', 'Food', 'Cafe', 'Health', 'Education', 'Shopping'];

  @override
  void initState() {
    super.initState();
    _loadLocation();
  }

  Future<void> _loadLocation() async {
    Position? position = await LocationService().getCurrentPosition();
    setState(() {
      _currentPosition = position;
      _isLoading = false;
    });
  }

  List<Place> get _filteredPlaces {
    List<Place> places = _selectedCategory == 'All'
        ? _allPlaces
        : _allPlaces.where((p) => p.category == _selectedCategory).toList();

    if (_currentPosition != null) {
      places.sort((a, b) => a
          .distanceTo(_currentPosition!.latitude, _currentPosition!.longitude)
          .compareTo(b.distanceTo(
              _currentPosition!.latitude, _currentPosition!.longitude)));
    }

    return places;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('สถานที่ใกล้เคียง')),
      body: Column(
        children: [
          // Category filter
          SizedBox(
            height: 48,
            child: ListView.builder(
              scrollDirection: Axis.horizontal,
              padding: const EdgeInsets.symmetric(horizontal: 12),
              itemCount: _categories.length,
              itemBuilder: (context, i) => Padding(
                padding: const EdgeInsets.only(right: 8),
                child: FilterChip(
                  label: Text(_categories[i]),
                  selected: _selectedCategory == _categories[i],
                  onSelected: (v) => setState(() => _selectedCategory = _categories[i]),
                ),
              ),
            ),
          ),
          // Places list
          Expanded(
            child: _isLoading
                ? const Center(child: CircularProgressIndicator())
                : ListView.builder(
                    itemCount: _filteredPlaces.length,
                    itemBuilder: (context, i) {
                      Place place = _filteredPlaces[i];
                      double? distance = _currentPosition != null
                          ? place.distanceTo(
                              _currentPosition!.latitude,
                              _currentPosition!.longitude,
                            )
                          : null;

                      return ListTile(
                        leading: CircleAvatar(
                          child: Text(place.category[0]),
                        ),
                        title: Text(place.name),
                        subtitle: Text('${place.category} • ⭐ ${place.rating}'),
                        trailing: distance != null
                            ? Text(distance < 1000
                                ? '${distance.toInt()}m'
                                : '${(distance / 1000).toStringAsFixed(1)}km')
                            : null,
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}
```

---

**← [Part 36 - Deep Linking & Notifications](part-36-deep-linking-notifications.md)**

**ต่อไป: [Part 38 - Video Player & Media →](part-38-video-media.md)**
