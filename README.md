# Car-REntal-Application
This application allows users to track and manage the location of vehicles in real-time. Built with Python (Django) on the backend, it handles user authentication, car data management, and location tracking through APIs. On the frontend, React.js is used to create a dynamic and responsive user interface that allows users to view and interact with the locations of cars on a map. The application uses JavaScript to make API calls for live location updates and provides an intuitive experience for tracking and managing vehicles.
-----------
To build a vehicle tracking application with Django as the backend and React.js for the frontend, we’ll need to break down the project into several parts:
Overview of the Project

    Backend (Django)
        User authentication (login, register).
        Vehicle data management.
        Vehicle location tracking via APIs.
    Frontend (React.js)
        Display vehicles and their real-time locations on a map.
        Use of APIs to fetch live data (e.g., Google Maps API or a similar service).
        Real-time updates (e.g., WebSocket, or periodic polling).

Step-by-Step Guide
1. Backend: Django Setup
1.1 Install Django and Dependencies

pip install django djangorestframework
pip install djangorestframework-simplejwt  # For JWT Authentication
pip install django-cors-headers  # To handle cross-origin requests from the React frontend

1.2 Set up Django Project and App

    Create a new Django project and an app.

django-admin startproject vehicle_tracker
cd vehicle_tracker
django-admin startapp tracker

    Add necessary apps to settings.py:

INSTALLED_APPS = [
    ...
    'rest_framework',
    'tracker',
    'corsheaders',
]

    Install CORS headers middleware:

MIDDLEWARE = [
    ...
    'corsheaders.middleware.CorsMiddleware',
]

    Allow cross-origin requests from React app:

CORS_ALLOWED_ORIGINS = [
    'http://localhost:3000',  # React's dev server
]

1.3 Create Models for Vehicles

Inside tracker/models.py, define the vehicle model:

from django.db import models

class Vehicle(models.Model):
    user = models.ForeignKey('auth.User', related_name='vehicles', on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    license_plate = models.CharField(max_length=20)
    current_lat = models.FloatField()
    current_lon = models.FloatField()
    last_updated = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.name

1.4 Create API Views

Inside tracker/views.py, create API views using Django Rest Framework (DRF) to manage vehicle data:

from rest_framework import viewsets
from .models import Vehicle
from .serializers import VehicleSerializer
from rest_framework.permissions import IsAuthenticated

class VehicleViewSet(viewsets.ModelViewSet):
    queryset = Vehicle.objects.all()
    serializer_class = VehicleSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        # Only return vehicles that belong to the authenticated user
        return self.queryset.filter(user=self.request.user)

1.5 Create Serializers for the Vehicle Model

Inside tracker/serializers.py, create a serializer to convert the model data to JSON:

from rest_framework import serializers
from .models import Vehicle

class VehicleSerializer(serializers.ModelSerializer):
    class Meta:
        model = Vehicle
        fields = ['id', 'name', 'license_plate', 'current_lat', 'current_lon', 'last_updated']

1.6 URL Routing

Inside tracker/urls.py, add routes for the API endpoints:

from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import VehicleViewSet

router = DefaultRouter()
router.register(r'vehicles', VehicleViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]

Then include this in your main vehicle_tracker/urls.py:

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('tracker.urls')),
]

1.7 JWT Authentication Setup

Inside settings.py, add JWT authentication settings:

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}

In urls.py, add routes for JWT authentication:

from rest_framework_simplejwt import views as jwt_views

urlpatterns = [
    path('api/token/', jwt_views.TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', jwt_views.TokenRefreshView.as_view(), name='token_refresh'),
]

1.8 Migrations and Run the Server

Run migrations to create the database schema for the vehicle model:

python manage.py makemigrations
python manage.py migrate

Finally, start the Django development server:

python manage.py runserver

2. Frontend: React.js Setup
2.1 Install React and Dependencies

    Set up a React project:

npx create-react-app vehicle-tracker-frontend
cd vehicle-tracker-frontend
npm install axios react-router-dom

    Install Google Maps React for displaying maps:

npm install @react-google-maps/api

2.2 Set up Authentication with JWT

For authentication, you will use JWT tokens to manage user login. In src/api.js, create functions to interact with the backend:

import axios from "axios";

const API_URL = "http://localhost:8000/api/";

// Login API call
export const login = (email, password) => {
  return axios.post(`${API_URL}token/`, {
    username: email,
    password: password,
  });
};

// Fetch vehicles
export const fetchVehicles = (token) => {
  return axios.get(`${API_URL}vehicles/`, {
    headers: { Authorization: `Bearer ${token}` },
  });
};

2.3 Create Map Component for Displaying Vehicle Locations

Use Google Maps or Leaflet.js to display the vehicles' current locations.

// src/components/VehicleMap.js
import React, { useState, useEffect } from "react";
import { useJsApiLoader, GoogleMap, Marker } from "@react-google-maps/api";
import { fetchVehicles } from "../api";

const containerStyle = {
  width: "100%",
  height: "500px",
};

const center = {
  lat: 37.7749, // Default to San Francisco
  lng: -122.4194,
};

const VehicleMap = ({ token }) => {
  const [vehicles, setVehicles] = useState([]);
  const { isLoaded } = useJsApiLoader({
    googleMapsApiKey: "YOUR_GOOGLE_MAPS_API_KEY",
  });

  useEffect(() => {
    if (token) {
      fetchVehicles(token)
        .then((response) => setVehicles(response.data))
        .catch((error) => console.error(error));
    }
  }, [token]);

  if (!isLoaded) return <div>Loading...</div>;

  return (
    <GoogleMap mapContainerStyle={containerStyle} center={center} zoom={12}>
      {vehicles.map((vehicle) => (
        <Marker
          key={vehicle.id}
          position={{
            lat: vehicle.current_lat,
            lng: vehicle.current_lon,
          }}
          title={vehicle.name}
        />
      ))}
    </GoogleMap>
  );
};

export default VehicleMap;

2.4 Create a Dashboard Page

// src/pages/Dashboard.js
import React, { useState } from "react";
import { login } from "../api";
import VehicleMap from "../components/VehicleMap";

const Dashboard = () => {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [token, setToken] = useState("");

  const handleLogin = () => {
    login(email, password).then((response) => {
      setToken(response.data.access);
    });
  };

  return (
    <div>
      {!token ? (
        <div>
          <h2>Login</h2>
          <input
            type="email"
            placeholder="Email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
          />
          <input
            type="password"
            placeholder="Password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
          />
          <button onClick={handleLogin}>Login</button>
        </div>
      ) : (
        <div>
          <h2>Vehicle Tracker Dashboard</h2>
          <VehicleMap token={token} />
        </div>
      )}
    </div>
  );
};

export default Dashboard;

2.5 Set up Routing in React

// src/App.js
import React from "react";
import { BrowserRouter as Router, Route, Switch } from "react-router-dom";
import Dashboard from "./pages/Dashboard";

function App() {
  return (
    <Router>
      <Switch>
        <Route path="/" component={Dashboard} />
      </Switch>
    </Router>
  );
}

export default App;

3. Real-Time Updates

To implement real-time updates for vehicle locations, you can:

    Use WebSockets or Server-Sent Events (SSE) to push location updates to the frontend.
    Alternatively, you can set up periodic polling from React to fetch vehicle locations at regular intervals.

4. Deployment

    Backend:
        Deploy the Django application on platforms like Heroku, AWS, or DigitalOcean.
        Use PostgreSQL or MySQL for the database.

    Frontend:
        Deploy the React app on Vercel or Netlify.
        Make sure to set the appropriate environment variables for Google Maps API key and API URL.

Conclusion

With this setup, you will have:

    Django backend to manage users, vehicles, and their locations.
    React.js frontend to display and interact with vehicle data in real time.
    A map integration for visualizing vehicles.

Let me know if you need additional features or further clarification!
