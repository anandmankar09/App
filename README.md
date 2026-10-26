import os
import eel
from modules import crypto_manager
from modules import db_manager

# Initialize Eel pointing to the web assets directory
eel.init('web')

@eel.expose
def verify_pin_bridge(pin):
    """Verifies the user PIN via MySQL database."""
    try:
        is_valid = db_manager.verify_quick_pin(pin)
        return {"status": "success", "authenticated": is_valid}
    except Exception as e:
        return {"status": "error", "message": str(e)}

@eel.expose
def encrypt_file_bridge(file_path, passphrase):
    """Encrypts a local file and persists its metadata to MySQL."""
    try:
        # Normalize file path
        cleaned_path = file_path.strip('\"').strip("\'")
        
        if not os.path.exists(cleaned_path):
            return {"status": "error", "message": "Source file does not exist."}
            
        # Execute AES-256 encryption via crypto_manager
        result = crypto_manager.encrypt_file(cleaned_path, passphrase)
        
        if result.get("success"):
            user_id = 1  # Default seed admin user
            db_manager.save_vault_file(
                user_id=user_id,
                original_filename=result.get("original_filename"),
                encrypted_filename=result.get("encrypted_filename"),
                file_size=result.get("file_size"),
                sha256_hash=result.get("sha256_hash")
            )
            return {"status": "success", "message": "File encrypted and recorded in database!", "data": result}
        else:
            return {"status": "error", "message": result.get("error", "Encryption failed")}
            
    except Exception as e:
        return {"status": "error", "message": str(e)}

@eel.expose
def get_vault_files_bridge():
    """Fetches previously encrypted files from MySQL for the dashboard table."""
    try:
        files = db_manager.get_user_vault_files(user_id=1)
        return {"status": "success", "files": files}
    except Exception as e:
        return {"status": "error", "message": str(e)}

if __name__ == '__main__':
    # Launch the desktop UI
    eel.start('index.html', size=(1280, 800), port=8080)
