```py

import azure.functions as func
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient
import json
import logging
from datetime import datetime, timedelta

app = func.FunctionApp() 

schedule_bp = func.Blueprint()


STARTSCHEDULETAG = "startSchedule"
STOPSCHEDULETAG = "stopSchedule"

CORS_HEADERS = {
    'Access-Control-Allow-Origin': '*',  
    'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type'
}

@schedule_bp.function_name(name="SetSchedule")
@schedule_bp.route(route="api/schedule", auth_level=func.AuthLevel.ANONYMOUS)
def set_schedule(req: func.HttpRequest) -> func.HttpResponse:
    try:
        vmData = req.get_json()

        vm_id_parts = vmData["id"].split("/")
        subscription_id = vm_id_parts[2]
        resource_group = vm_id_parts[4]
        vm_name = vm_id_parts[8]
        compute_client = ComputeManagementClient(
            credential=DefaultAzureCredential(exclude_environment_credential=True),
            subscription_id=subscription_id
        )

        logging.info(f"Starting VM: {vm_name}")
        start_vm_operation = compute_client.virtual_machines.begin_start(
            resource_group_name=resource_group,
            vm_name=vm_name
        )
        start_vm_operation.wait()  # Wait for VM to turn on

       
        start_time = datetime.utcnow()
        stop_time = start_time + timedelta(hours=8)

     
        start_schedule = f"{start_time.minute} {start_time.hour} * * *"
        stop_schedule = f"{stop_time.minute} {stop_time.hour} * * *"

      
        vm_instance = compute_client.virtual_machines.get(resource_group_name=resource_group, vm_name=vm_name)
        tags = vm_instance.tags or {}  # Get existing tags, or initialize an empty dictionary
        tags[STARTSCHEDULETAG] = start_schedule
        tags[STOPSCHEDULETAG] = stop_schedule

      
        logging.info(f"Setting auto-shutdown for {vm_name} at {stop_time}")
        update_operation = compute_client.virtual_machines.begin_create_or_update(
            resource_group_name=resource_group,
            vm_name=vm_name,
            parameters={"location": vm_instance.location, "tags": tags}
        )
        update_operation.wait() 

      
        return func.HttpResponse(
            "VM started successfully and will shut down in 8 hours.",
            status_code=200,
            headers=CORS_HEADERS
        )

    except Exception as e:
        logging.error(f"Error in set_schedule function: {str(e)}")
    
        return func.HttpResponse(
            f"Failed to start VM: {str(e)}",
            status_code=500,
            headers=CORS_HEADERS
        )


app.register_blueprint(schedule_bp)







```
