
import azure.functions as func

from azure.iot.hub import IoTHubRegistryManager

from azure.cosmos import CosmosClient

import numpy as np

from datetime import datetime

import json

# Configuration

CONNECTION_STRING = "YOUR_IOT_HUB_CONNECTION_STRING"

COSMOS_URI = "YOUR_COSMOS_DB_URI"

COSMOS_KEY = "YOUR_COSMOS_DB_KEY"



class TrafficManagementSystem:

    def __init__(self):
        self.iot_registry = IoTHubRegistryManager(CONNECTION_STRING)
        self.cosmos_client = CosmosClient(COSMOS_URI, COSMOS_KEY)
        self.database = self.cosmos_client.get_database_client("TrafficDB")
        self.container = self.database.get_container_client("TrafficData")
        
    def process_sensor_data(self, sensor_data):
        """Process incoming sensor data and predict traffic patterns"""
        traffic_density = self._calculate_traffic_density(sensor_data)
        predicted_wait_time = self._predict_wait_time(traffic_density)
        signal_timing = self._calculate_signal_timing(predicted_wait_time)
        
        return {
            "traffic_density": traffic_density,
            "wait_time": predicted_wait_time,
            "signal_timing": signal_timing
        }
    
    def _calculate_traffic_density(self, sensor_data):
        """Calculate traffic density from sensor data"""
        # Simple density calculation based on vehicle count and road capacity
        vehicle_count = sensor_data.get("vehicle_count", 0)
        road_capacity = sensor_data.get("road_capacity", 100)
        return min((vehicle_count / road_capacity) * 100, 100)
    
    def _predict_wait_time(self, traffic_density):
        """Predict wait time based on traffic density"""
        # Linear scaling between 30-150 seconds based on density
        min_wait = 30
        max_wait = 150
        return min_wait + (traffic_density / 100) * (max_wait - min_wait)
    
    def _calculate_signal_timing(self, wait_time):
        """Calculate optimal signal timing"""
        green_time = max(min(wait_time, 150), 30)  # Bounded between 30-150 seconds
        yellow_time = 5
        return {
            "green": green_time,
            "yellow": yellow_time,
            "red": green_time + yellow_time
        }
    
    async def store_traffic_data(self, junction_id, traffic_data):
        """Store traffic data in Cosmos DB"""
        document = {
            "id": f"{junction_id}-{datetime.utcnow().isoformat()}",
            "junction_id": junction_id,
            "timestamp": datetime.utcnow().isoformat(),
            "traffic_data": traffic_data
        }
        await self.container.create_item(document)
        
# Azure Function HTTP trigger

async def main(req: func.HttpRequest) -> func.HttpResponse:

    try:
        traffic_system = TrafficManagementSystem()
        
        # Get sensor data from request
        sensor_data = req.get_json()
        junction_id = sensor_data.get("junction_id")
        
        # Process traffic data
        result = traffic_system.process_sensor_data(sensor_data)
        
        # Store results
        await traffic_system.store_traffic_data(junction_id, result)
        
        return func.HttpResponse(
            json.dumps(result),
            mimetype="application/json",
            status_code=200
        )
    except Exception as e:
        return func.HttpResponse(
            f"Error processing request: {str(e)}",
            status_code=500
        )
