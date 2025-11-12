# routes/api_routes.py
 
import time
import random
import uuid
import requests
import json
from flask import Blueprint, jsonify, request, current_app, Response, send_from_directory, g
 
# Make sure your History model is importable if you use .to_dict()
from services.db_storage import DBStorage, TaskIdAlreadyExistsError, History
from services.redis_storage import RedisStorage
from services.hybrid_storage import HybridStorage
 
MOCKED_TASK_STORE = {}
 
def bp_factory(redis_client_instance=None):
 bp = Blueprint("api", __name__, url_prefix="/api/core")
 
 def get_storage():
  mode = current_app.config["STORAGE_MODE"]
  if mode == "redis": return RedisStorage(redis_client_instance)
  elif mode == "db": return DBStorage()
  else: return HybridStorage(redis_client_instance)
 
 # Service configuration
 SERVICES_CONFIG = {
  "Apps": {
   "OKTA": {
    "description": "Manage OKTA applications and permissions.",
    "metadata_url": "okta-spec.json",
    "status": "OK",
    "message": "Service is operational.",
    "tags": ["identity", "auth", "sso"],
    "category": "Cloud",
    "disabled": False
   },
   "SNOW": {
    "description": "Manage user roles via a dedicated UI.",
    "metadata_url": "snow-spec.json",
    "status": "OK",
    "message": "Service is operational.",
    "tags": ["ticketing", "iam", "roles"],
    "category": "Cloud",
    "disabled": False
   },
   "WinDNS": {
    "description": "Manage DNS records for specific domains.",
    "metadata_url": "windns-spec.json",
    "status": "OK",
    "message": "Service is operational.",
    "tags": ["networking", "dns"],
    "category": "Network",
    "disabled": False
   },
   "STS": {
    "description": "Endpoints for modifying user account information.",
    "metadata_url": "sts-spec.json",
    "status": "ERROR",
    "message": "Service is experiencing intermittent issues.",
    "tags": ["accounts", "users"],
    "category": "Network",
    "disabled": False
   },
   "AD": {
    "description": "Endpoints for managing AD groups",
    "metadata_url": "ad-spec.json",
    "status": "OK",
    "message": "Service is operational.",
    "tags": ["users", "groups"],
    "category": "Network",
    "disabled": False
   },
   "Terraform": {
    "description": "Provision and Manage of Infrastructure resources.",
    "metadata_url": "terraform-spec.json",
    "status": "ERROR",
    "message": "Service is experiencing intermittent issues.",
    "tags": ["infrastructure", "cloud"],
    "category": "Cloud",
    "disabled": False
   },
   "ORIGIN": {
    "description": "Endpoints for managing AURA applications",
    "metadata_url": "origin-spec.json",
    "status": "OK",
    "message": "Service is operational.",
    "tags": ["cloud", "network"],
    "category": "Network",
    "disabled": False
   },
   "VMSB": {
    "description": "Use GenAI to clone your repositories, access files, generate vulnerability reports, and securely manage projects.",
    "metadata_url": None,
    "status": "OK",
    "message": "Service is operational.",
    "ui_type": "iframe",
    "get_iframe_url": "http://127.0.0.1:5001/api/login",
    "tags": ["genai", "vulnerability", "code", "security"],
    "category": "Scanners",
    "disabled": True
   },
   "WIDGETS": {
    "description": "Demo service showcasing custom UI components.",
    "metadata_url": "custom-ui-spec.json",
    "status": "OK",
    "message": "Service is operational.",
    "tags": ["demo", "ui"],
    "category": "Demo",
    "disabled": False
   },
   "CLOUDFLARE": {
    "description": "Manage Cloudflare DNS records, firewall rules, and CDN configurations with advanced automation",
    "metadata_url": "cloudflare-spec.json",
    "status": "OK",
    "message": "Service is operational.",
    "tags": ["cdn", "dns", "firewall", "security", "networking"],
    "category": "Demo",
    "disabled": False,
    # NEW: Special backend URL for Cloudflare mock service
    "mock_backend_url": "http://localhost:5002"
   }
  }
 }
 SPEC_DIRECTORY = 'specs'
 
 @bp.route('/services')
 def get_services():
  return jsonify(SERVICES_CONFIG)
 
 @bp.route('/info/<path:spec_path>')
 def get_service_spec(spec_path):
  service_info = next((cfg for _, cfg in SERVICES_CONFIG['Apps'].items() if cfg.get('metadata_url') == spec_path), None)
  if not service_info:
   return jsonify({"error": "No service is configured for this specification URL"}), 404
  try:
   return send_from_directory(SPEC_DIRECTORY, spec_path)
  except FileNotFoundError:
   return jsonify({"error": f"Specification file '{spec_path}' not found on server."}), 404
 
 @bp.route('/execute/<path:application_name>/<path:endpoint>', methods=['GET', 'POST', 'PUT', 'DELETE', 'PATCH'])
 def handle_service_api(application_name, endpoint):
  if application_name not in SERVICES_CONFIG['Apps']:
   return jsonify({"error": f"Application '{application_name}' is not configured."}), 404
 
  storage = get_storage()
  request_data = request.get_json(silent=True) or {}
   
  log_payload = {
   "user_id": g.user_id,
   "application": application_name,
   "target_url": f"/{endpoint}",
   "request_body": request_data,
   "status": "PENDING",
  }
   
  # --- NEW: SPECIAL HANDLING FOR CLOUDFLARE ---
  service_config = SERVICES_CONFIG['Apps'][application_name]
  cloudflare_mock_url = service_config.get('mock_backend_url')
   
  if cloudflare_mock_url and application_name == 'CLOUDFLARE':
   # CLOUDFLARE uses its dedicated mock service even in mock mode
   record_id = storage.save_request(log_payload)
    
   try:
    target_url = f"{cloudflare_mock_url}/{endpoint}"
    headers = {k: v for k, v in request.headers if k.lower() != 'host'}
     
    resp = requests.request(
     method=request.method,
     url=target_url,
     headers=headers,
     params=request.args.to_dict(),
     json=request_data if request.method != 'GET' else None,
     data=None if request.method == 'GET' and request_data else None,
     timeout=10
    )
     
    response_body = {}
    try:
     response_body = resp.json()
    except json.JSONDecodeError:
     response_body = {"raw_content": resp.text[:1000]}
     
    update_payload = {
     "response_body": response_body,
     "status": "SUCCESS" if 200 <= resp.status_code < 300 else "ERROR"
    }
    storage.update_response(record_id, update_payload)
     
    # Return the response from Cloudflare mock service
    return jsonify(response_body), resp.status_code
     
   except requests.exceptions.RequestException as e:
    error_payload = {
     "response_body": {"error": "Failed to connect to Cloudflare mock service", "details": str(e)},
     "status": "ERROR"
    }
    storage.update_response(record_id, error_payload)
    return jsonify(error_payload["response_body"]), 502
   
  # --- GENERIC MOCK MODE FLOW (for non-Cloudflare services) ---
  if current_app.config["MOCK_API_ENABLED"]:
   is_async = request.method != 'GET' and random.random() < 0.5
    
   if is_async:
    task_id = str(uuid.uuid4())
    log_payload["task_id"] = task_id
    log_payload["poll_url"] = f"/api/core/task/{task_id}"
    
   record_id = storage.save_request(log_payload)
 
   if is_async:
    MOCKED_TASK_STORE[task_id] = {'status': 'PENDING', 'start_time': time.time()}
    response_body = {
     "message": "Task accepted for processing.",
     "response": {
      "task_id": task_id,
      "task_endpoint": log_payload["poll_url"],
      "message": "Poll the task endpoint to get the status."
     }
    }
    return jsonify(response_body), 202
   else:
    time.sleep(0.5)
    sync_response = {"success": True, "message": f"Mock operation '{endpoint}' executed synchronously."}
    storage.update_response(record_id, {"response_body": sync_response, "status": "SUCCESS"})
    return jsonify(sync_response)
 
  # --- PROXY MODE FLOW (if mock is disabled) ---
  else:
   base_url = current_app.config["REAL_BACKEND_URL"]
   if not base_url:
    return jsonify({"error": "Proxy mode enabled but REAL_BACKEND_URL not configured."}), 500
    
   record_id = storage.save_request(log_payload)
    
   try:
    target_url = f"{base_url}/{endpoint}"
    headers = {k: v for k, v in request.headers if k.lower() != 'host'}
    resp = requests.request(
     method=request.method,
     url=target_url,
     headers=headers,
     params=request.args.to_dict(),
     data=request.get_data(),
     timeout=30
    )
 
    if resp.status_code == 202:
     our_task_id = str(uuid.uuid4())
     our_poll_url = f"/api/core/task/{our_task_id}"
 
     update_payload = {
      "task_id": our_task_id,
      "poll_url": our_poll_url,
      "status": "PENDING",
      "response_body": {
       "message": "Request accepted by backend, polling initiated.",
       "backend_response": resp.json()
      }
     }
     storage.update_response(record_id, update_payload)
 
     frontend_response = {
      "message": "Task accepted for processing.",
      "response": {
       "task_id": our_task_id,
       "task_endpoint": our_poll_url
      }
     }
     return jsonify(frontend_response), 202
 
    else:
     response_body = {}
     try:
      response_body = resp.json()
     except json.JSONDecodeError:
      response_body = {"raw_content": resp.text[:1000]}
      
     update_payload = {
      "response_body": response_body,
      "status": "SUCCESS" if 200 <= resp.status_code < 300 else "ERROR"
     }
     storage.update_response(record_id, update_payload)
 
     excluded_headers = ['content-encoding', 'content-length', 'transfer-encoding', 'connection']
     response_headers = [(n, v) for (n, v) in resp.raw.headers.items() if n.lower() not in excluded_headers]
     return Response(resp.content, resp.status_code, response_headers)
 
   except requests.exceptions.RequestException as e:
    error_payload = {
     "response_body": {"error": "Proxy request failed", "details": str(e)},
     "status": "ERROR"
    }
    storage.update_response(record_id, error_payload)
    return jsonify(error_payload["response_body"]), 502
 
 @bp.route('/task/<task_id>')
 def get_task_status(task_id):
  if not current_app.config["MOCK_API_ENABLED"]:
   return jsonify({"error": "Live task polling is not implemented. This endpoint is for mock mode only."}), 400
 
  task = MOCKED_TASK_STORE.get(task_id)
  if not task:
   return jsonify({"error": "Task not found in active mock store"}), 404
 
  storage = get_storage()
  if task['status'] in ['SUCCESS', 'ERROR']:
   final_record = storage.get_history({"task_id": task_id}, 1, 1)
   if final_record and isinstance(final_record[0], History):
    return jsonify(final_record[0].to_dict())
   return jsonify({"status": task['status'], "result": task.get('result')})
 
  elapsed_time = time.time() - task['start_time']
   
  if elapsed_time > 15:
   if random.random() > 0.2:
    task['status'] = 'SUCCESS'
    task['result'] = {"message": "Task completed successfully!", "final_output": "ok"}
    storage.update_response(task_id, {"response_body": task['result'], "status": "SUCCESS"})
   else:
    task['status'] = 'ERROR'
    task['result'] = {"error": "Task failed due to a simulated error."}
    storage.update_response(task_id, {"response_body": task['result'], "status": "ERROR"})
    
   final_record = storage.get_history({"task_id": task_id}, 1, 1)
   if final_record and isinstance(final_record[0], History):
    return jsonify(final_record[0].to_dict())
   return jsonify({"status": task['status'], "result": task.get('result')})
 
  elif elapsed_time > 5:
   task['status'] = 'RUNNING'
 
  return jsonify({"status": task['status'], "task_id": task_id})
 
 return bp
