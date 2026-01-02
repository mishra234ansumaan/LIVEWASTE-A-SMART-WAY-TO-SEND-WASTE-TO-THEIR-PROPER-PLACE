# LIVEWASTE-A-SMART-WAY-TO-SEND-WASTE-TO-THEIR-PROPER-PLACE
Basically the app LIVEWASTE leverages Google AI and maps to identify garbage from images and display cleanliness status live on a city map. Just like traffic waste becomes visible , trackable , and actionable in real time. 
I am writing the programming codes I have used for this.

// Hackathon prototype entry point for LIVEWASTE
// Focus: user flow clarity, real-time visualization, and demo stability

import React, { useState, useEffect } from "react";
import { Users, Truck, Loader2 } from "lucide-react";
import { Button } from "./components/ui/button";
import { Card } from "./components/ui/card";
import { CitizenView } from "./components/CitizenView";
import { WorkerView } from "./components/WorkerView";
import { LocationPermissionModal } from "./components/LocationPermissionModal";
import { Toaster } from "./components/ui/sonner";
import { toast } from "sonner@2.0.3";
import {
  projectId,
  publicAnonKey,
} from "./utils/supabase/info";

type UserRole = "citizen" | "worker" | null;

export default function App() {
  const [userRole, setUserRole] = useState<UserRole>(null);
  const [bins, setBins] = useState([]);
  const [trucks, setTrucks] = useState([]);
  const [reports, setReports] = useState([]);
  const [predictions, setPredictions] = useState([]);
  const [stats, setStats] = useState({
    totalBins: 0,
    redZones: 0,
    yellowZones: 0,
    greenZones: 0,
    pendingReports: 0,
    completedToday: 0,
  });
  const [isLoading, setIsLoading] = useState(true);
  const [isInitialized, setIsInitialized] = useState(false);
  const [showLocationModal, setShowLocationModal] =
    useState(false);
  // Set default location to Sambalpur, Odisha, India immediately
  const [userLocation, setUserLocation] = useState<{
    lat: number;
    lng: number;
  }>({ lat: 21.4669, lng: 83.9812 });
  const [pendingRole, setPendingRole] =
    useState<UserRole>(null);

  const baseUrl = `https://${projectId}.supabase.co/functions/v1/make-server-1c07b05c`;

  useEffect(() => {
    initializeApp();
  }, []);

  useEffect(() => {
    if (userRole) {
      // Refresh data every 10 seconds for real-time updates
      const interval = setInterval(() => {
        fetchData();
      }, 10000);

      return () => clearInterval(interval);
    }
  }, [userRole]);

  const initializeApp = async () => {
    try {
      // Initialize demo data
      await fetch(`${baseUrl}/init-demo`, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${publicAnonKey}`,
        },
      });

      setIsInitialized(true);
      await fetchData();
    } catch (error) {
      console.error("Error initializing app:", error);
    } finally {
      setIsLoading(false);
    }
  };

  const fetchData = async () => {
    try {
      const [
        binsRes,
        trucksRes,
        reportsRes,
        predictionsRes,
        statsRes,
      ] = await Promise.all([
        fetch(`${baseUrl}/bins`, {
          headers: { Authorization: `Bearer ${publicAnonKey}` },
        }),
        fetch(`${baseUrl}/trucks`, {
          headers: { Authorization: `Bearer ${publicAnonKey}` },
        }),
        fetch(`${baseUrl}/reports`, {
          headers: { Authorization: `Bearer ${publicAnonKey}` },
        }),
        fetch(`${baseUrl}/predictions`, {
          headers: { Authorization: `Bearer ${publicAnonKey}` },
        }),
        fetch(`${baseUrl}/stats`, {
          headers: { Authorization: `Bearer ${publicAnonKey}` },
        }),
      ]);

      const binsData = await binsRes.json();
      const trucksData = await trucksRes.json();
      const reportsData = await reportsRes.json();
      const predictionsData = await predictionsRes.json();
      const statsData = await statsRes.json();

      setBins(binsData.bins || []);
      setTrucks(trucksData.trucks || []);
      setReports(reportsData.reports || []);
      setPredictions(predictionsData.predictions || []);
      setStats(
        statsData.stats || {
          totalBins: 0,
          redZones: 0,
          yellowZones: 0,
          greenZones: 0,
          pendingReports: 0,
          completedToday: 0,
        },
      );
    } catch (error) {
      console.error("Error fetching data:", error);
    }
  };

  const handleReport = async (report: any) => {
    try {
      const response = await fetch(`${baseUrl}/reports`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${publicAnonKey}`,
        },
        body: JSON.stringify(report),
      });

      if (response.ok) {
        await fetchData();
      }
    } catch (error) {
      console.error("Error creating report:", error);
    }
  };

  const handlePickupConfirm = async (pickup: any) => {
    try {
      const response = await fetch(
        `${baseUrl}/pickups/confirm`,
        {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
            Authorization: `Bearer ${publicAnonKey}`,
          },
          body: JSON.stringify(pickup),
        },
      );

      if (response.ok) {
        await fetchData();
      }
    } catch (error) {
      console.error("Error confirming pickup:", error);
    }
  };

  const handleLocationPermission = (
    permissionGranted: boolean,
  ) => {
    setShowLocationModal(false);

    // Always set the role immediately so map shows right away
    if (pendingRole) {
      setUserRole(pendingRole);
      setPendingRole(null);
    }

    if (permissionGranted) {
      // Try to get actual location in background (won't block map from showing)
      if ("geolocation" in navigator) {
        toast.info("Attempting to detect your location...");
        navigator.geolocation.getCurrentPosition(
          (position) => {
            const location = {
              lat: position.coords.latitude,
              lng: position.coords.longitude,
            };
            setUserLocation(location);
            toast.success(
              `✓ Location updated: ${location.lat.toFixed(4)}°N, ${location.lng.toFixed(4)}°E`,
            );
          },
          (error) => {
            console.error(
              "Location error:",
              error.message || error.code || error,
            );
            // Silently use default location without showing error message
          },
          {
            enableHighAccuracy: true,
            timeout: 5000,
            maximumAge: 0,
          },
        );
      } else {
        toast.info("Using default location: Sambalpur, Odisha");
      }
    } else {
      toast.info("Using default location: Sambalpur, Odisha");
    }
  };

  if (isLoading) {
    return (
      <div className="h-screen flex items-center justify-center bg-gradient-to-br from-emerald-50 to-teal-50">
        <div className="text-center">
          <Loader2 className="w-12 h-12 animate-spin mx-auto mb-4 text-emerald-600" />
          <h2 className="text-xl mb-2 text-gray-800">
            Loading LIVEWASTE
          </h2>
          <p className="text-gray-600 text-sm">
            Initializing real-time waste tracking...
          </p>
        </div>
      </div>
    );
  }

  if (!userRole) {
    return (
      <div className="h-screen flex flex-col bg-gradient-to-br from-emerald-50 via-teal-50 to-cyan-50 animate-fade-in">
        <Toaster />

        {/* Location Permission Modal */}
        {showLocationModal && (
          <LocationPermissionModal
            onGrant={() => handleLocationPermission(true)}
            onDeny={() => handleLocationPermission(false)}
          />
        )}

        {/* Header */}
        <div className="bg-gradient-to-r from-emerald-600 via-teal-600 to-cyan-600 text-white p-8 text-center shadow-2xl relative overflow-hidden">
          <div className="absolute inset-0 bg-[url('data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iNjAiIGhlaWdodD0iNjAiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+PGRlZnM+PHBhdHRlcm4gaWQ9ImdyaWQiIHdpZHRoPSI2MCIgaGVpZ2h0PSI2MCIgcGF0dGVyblVuaXRzPSJ1c2VyU3BhY2VPblVzZSI+PHBhdGggZD0iTSAxMCAwIEwgMCAwIDAgMTAiIGZpbGw9Im5vbmUiIHN0cm9rZT0id2hpdGUiIHN0cm9rZS13aWR0aD0iMC41IiBvcGFjaXR5PSIwLjEiLz48L3BhdHRlcm4+PC9kZWZzPjxyZWN0IHdpZHRoPSIxMDAlIiBoZWlnaHQ9IjEwMCUiIGZpbGw9InVybCgjZ3JpZCkiLz48L3N2Zz4=')] opacity-20"></div>
          <div className="relative z-10">
            <div className="text-7xl mb-4 animate-bounce-in">
              🗑️
            </div>
            <h1 className="text-4xl mb-3 font-extrabold tracking-tight animate-slide-in-left">
              LIVEWASTE
            </h1>
            <p
              className="text-emerald-100 text-lg animate-slide-in-left"
              style={{ animationDelay: "0.1s" }}
            >
              Real-time City Waste Flow Tracking
            </p>
            <p
              className="text-emerald-200 mt-2 flex items-center justify-center gap-2 animate-slide-in-left"
              style={{ animationDelay: "0.2s" }}
            >
              <span className="inline-block w-2 h-2 bg-emerald-300 rounded-full animate-pulse"></span>
              Like Google Traffic, but for Waste
            </p>
          </div>
        </div>

        {/* Features Banner */}
        <div className="bg-white py-5 px-6 shadow-md border-b-2 border-emerald-100">
          <div className="flex items-center justify-center gap-6 text-xs text-gray-600 overflow-x-auto">
            <div
              className="flex items-center gap-2 whitespace-nowrap animate-slide-in"
              style={{ animationDelay: "0.3s" }}
            >
              <div className="w-3 h-3 rounded-full bg-gradient-to-br from-red-500 to-red-600 shadow-lg animate-pulse"></div>
              <span className="font-medium">Red Zones</span>
            </div>
            <div
              className="flex items-center gap-2 whitespace-nowrap animate-slide-in"
              style={{ animationDelay: "0.4s" }}
            >
              <div className="w-3 h-3 rounded-full bg-gradient-to-br from-yellow-400 to-yellow-500 shadow-lg"></div>
              <span className="font-medium">Yellow Zones</span>
            </div>
            <div
              className="flex items-center gap-2 whitespace-nowrap animate-slide-in"
              style={{ animationDelay: "0.5s" }}
            >
              <div className="w-3 h-3 rounded-full bg-gradient-to-br from-green-500 to-green-600 shadow-lg"></div>
              <span className="font-medium">Green Zones</span>
            </div>
            <div
              className="flex items-center gap-2 whitespace-nowrap animate-slide-in"
              style={{ animationDelay: "0.6s" }}
            >
              <span className="text-lg">🚛</span>
              <span className="font-medium">Live Trucks</span>
            </div>
            <div
              className="flex items-center gap-2 whitespace-nowrap animate-slide-in"
              style={{ animationDelay: "0.7s" }}
            >
              <span className="text-lg">🤖</span>
              <span className="font-medium">
                AI Predictions
              </span>
            </div>
          </div>
        </div>

        {/* Role Selection */}
        <div className="flex-1 flex items-center justify-center p-6 overflow-y-auto">
          <div className="w-full max-w-md">
            <h2 className="text-3xl text-center mb-3 text-gray-800 font-bold animate-slide-in">
              Select Your Role
            </h2>
            <p
              className="text-center text-gray-600 mb-10 animate-slide-in"
              style={{ animationDelay: "0.1s" }}
            >
              Choose how you want to contribute to a cleaner
              city
            </p>

            <div className="space-y-5">
              <Card
                className="p-6 cursor-pointer hover:shadow-2xl transition-all duration-300 hover:scale-105 border-2 hover:border-emerald-500 bg-white animate-bounce-in relative overflow-hidden"
                style={{ animationDelay: "0.2s" }}
                onClick={() => {
                  setPendingRole("citizen");
                  setShowLocationModal(true);
                }}
              >
                <div className="absolute top-0 right-0 w-32 h-32 bg-gradient-to-br from-emerald-500/10 to-transparent rounded-full -mr-16 -mt-16"></div>
                <div className="flex items-start gap-4 relative z-10">
                  <div className="w-20 h-20 rounded-2xl bg-gradient-to-br from-emerald-500 via-teal-500 to-cyan-500 flex items-center justify-center flex-shrink-0 shadow-xl">
                    <Users className="w-10 h-10 text-white" />
                  </div>
                  <div className="flex-1">
                    <h3 className="mb-3 text-xl font-bold text-gray-800">
                      I'm a Citizen
                    </h3>
                    <p className="text-sm text-gray-600 mb-4">
                      Report waste issues and help keep your
                      neighborhood clean
                    </p>
                    <div className="space-y-2 text-xs text-gray-500">
                      <div className="flex items-center gap-2">
                        <span className="text-emerald-600">
                          ✓
                        </span>
                        <span>Scan waste with AI camera</span>
                      </div>
                      <div className="flex items-center gap-2">
                        <span className="text-emerald-600">
                          ✓
                        </span>
                        <span>View live waste heat map</span>
                      </div>
                      <div className="flex items-center gap-2">
                        <span className="text-emerald-600">
                          ✓
                        </span>
                        <span>1-tap overflow reporting</span>
                      </div>
                    </div>
                  </div>
                </div>
              </Card>

              <Card
                className="p-6 cursor-pointer hover:shadow-2xl transition-all duration-300 hover:scale-105 border-2 hover:border-blue-500 bg-white animate-bounce-in relative overflow-hidden"
                style={{ animationDelay: "0.3s" }}
                onClick={() => {
                  setPendingRole("worker");
                  setShowLocationModal(true);
                }}
              >
                <div className="absolute top-0 right-0 w-32 h-32 bg-gradient-to-br from-blue-500/10 to-transparent rounded-full -mr-16 -mt-16"></div>
                <div className="flex items-start gap-4 relative z-10">
                  <div className="w-20 h-20 rounded-2xl bg-gradient-to-br from-blue-500 via-cyan-500 to-teal-500 flex items-center justify-center flex-shrink-0 shadow-xl">
                    <Truck className="w-10 h-10 text-white" />
                  </div>
                  <div className="flex-1">
                    <h3 className="mb-3 text-xl font-bold text-gray-800">
                      I'm a Worker
                    </h3>
                    <p className="text-sm text-gray-600 mb-4">
                      Manage routes and confirm waste pickups
                    </p>
                    <div className="space-y-2 text-xs text-gray-500">
                      <div className="flex items-center gap-2">
                        <span className="text-blue-600">✓</span>
                        <span>View optimized routes</span>
                      </div>
                      <div className="flex items-center gap-2">
                        <span className="text-blue-600">✓</span>
                        <span>Confirm pickup completion</span>
                      </div>
                      <div className="flex items-center gap-2">
                        <span className="text-blue-600">✓</span>
                        <span>Get missed area alerts</span>
                      </div>
                    </div>
                  </div>
                </div>
              </Card>
            </div>
          </div>
        </div>

        {/* Tech Stack Footer */}
        <div className="bg-white border-t-2 border-gray-200 py-5 px-6 shadow-lg">
          <div className="text-center text-xs text-gray-500">
            <p className="mb-3 font-semibold text-gray-700 uppercase tracking-wide">
              Powered by:
            </p>
            <div className="flex items-center justify-center gap-3 flex-wrap">
              <span className="bg-gradient-to-r from-emerald-100 to-teal-100 text-emerald-700 px-4 py-2 rounded-full font-medium shadow-sm">
                Google Maps
              </span>
              <span className="bg-gradient-to-r from-blue-100 to-cyan-100 text-blue-700 px-4 py-2 rounded-full font-medium shadow-sm">
                ML Kit Vision AI
              </span>
              <span className="bg-gradient-to-r from-purple-100 to-pink-100 text-purple-700 px-4 py-2 rounded-full font-medium shadow-sm">
                Vertex AI
              </span>
              <span className="bg-gradient-to-r from-orange-100 to-amber-100 text-orange-700 px-4 py-2 rounded-full font-medium shadow-sm">
                Real-time Updates
              </span>
            </div>
          </div>
        </div>
      </div>
    );
  }

  if (userRole === "citizen") {
    return (
      <>
        \n <Toaster />
        <CitizenView
          bins={bins}
          trucks={trucks}
          reports={reports}
          stats={stats}
          onReport={handleReport}
          onRefresh={fetchData}
          onBack={() => setUserRole(null)}
          userLocation={userLocation}
        />
      </>
    );
  }

  if (userRole === "worker") {
    return (
      <>
        \n <Toaster />
        <WorkerView
          bins={bins}
          trucks={trucks}
          reports={reports}
          predictions={predictions}
          onPickupConfirm={handlePickupConfirm}
          onRefresh={fetchData}
          onBack={() => setUserRole(null)}
          userLocation={userLocation}
        />
      </>
    );
  }

  return null;
}

// Role & UI state
const [userRole, setUserRole] = useState<UserRole>(null);
const [isLoading, setIsLoading] = useState(true);
const [isInitialized, setIsInitialized] = useState(false);
const [showLocationModal, setShowLocationModal] = useState(false);
const [pendingRole, setPendingRole] = useState<UserRole>(null);


// Location state (default: Sambalpur, Odisha)
const [userLocation, setUserLocation] = useState<{ lat: number; lng: number }>({
  lat: 21.4669,
  lng: 83.9812,
});
