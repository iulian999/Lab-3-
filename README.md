# Lab-3-

import os
import time
import threading
import datetime
import mimetypes
from abc import ABC, abstractmethod
from collections import defaultdict


class FileInfo(ABC):
    def __init__(self, filename):
        self.filename = filename

    @abstractmethod
    def get_info(self):
        pass

class ImageFileInfo(FileInfo):
    def get_info(self):
        pass

class TextFileInfo(FileInfo):
    def get_info(self):
        pass

class PythonFileInfo(FileInfo):
    def get_info(self):
        pass

class JavaFileInfo(FileInfo):
    def get_info(self):
        pass

class FileMonitor:
    def __init__(self, folder_path):
        self.folder_path = folder_path
        self.snapshot_time = None
        self.file_info = defaultdict(dict)
        self.instance = FileInfo
        self.file_type = None
        self.lock = threading.Lock()

    def commit(self):
        self.snapshot_time = datetime.datetime.now()

    def get_file_instance(self, filename):
        file_path = os.path.join(self.folder_path, filename)
        if os.path.exists(file_path):
            file_type = mimetypes.guess_type(file_path)[0]
            self.file_type = file_type
            if file_type in ('image/png', 'image/jpeg'):
                instance = ImageFileInfo
            elif file_type == 'text/plain':
                instance = TextFileInfo
            elif file_type == 'text/x-python' or file_type == 'application/x-python-code':
                instance = PythonFileInfo
            elif file_type == 'text/x-java-source':
                instance = JavaFileInfo
            else:
                instance = FileInfo
            return instance
        else:
            return None

    def monitor_folder(self):
        with self.lock:
            for filename in os.listdir(self.folder_path):
                if os.path.isfile(os.path.join(self.folder_path, filename)):
                    Instance = self.get_file_instance(filename)
                    if Instance:
                        self.file_info[filename] = Instance(filename).get_info(self.folder_path)

    def info(self, filename):
        with self.lock:
            Instance = self.get_file_instance(filename)
            if Instance:
                file_instance = Instance(filename)
                file_info = file_instance.get_info()
                if file_info:
                    print(f"Name: {filename}")
                    print(f"Type: {self.file_type or file_instance.__class__.__name__}")
                    creation_time = os.path.getctime(os.path.join(self.folder_path, filename))
                    modification_time = os.path.getmtime(os.path.join(self.folder_path, filename))
                    print(f"Created: {datetime.datetime.fromtimestamp(creation_time)}")
                    print(f"Updated: {datetime.datetime.fromtimestamp(modification_time)}")

                    if isinstance(file_instance, ImageFileInfo):
                        print(f"Image Size: {file_info} bytes")
                    elif isinstance(file_instance, TextFileInfo):
                        print(f"Line Count: {file_info[0]}")
                        print(f"Word Count: {file_info[1]}")
                        print(f"Character Count: {file_info[2]}")
                    elif isinstance(file_instance, PythonFileInfo) or isinstance(file_instance, JavaFileInfo):
                        print(f"Line Count: {file_info[0]}")
                        print(f"Class Count: {file_info[1]}")
                        print(f"Method Count: {file_info[2]}")
                else:
                    print("File not found or does not exist.")
            else:
                print("File not found or does not exist.")

    def status(self):
        with self.lock:
            print("Status:")
            if not self.snapshot_time:
                print("No snapshot taken. Use 'commit' to take a snapshot.")
                return

            current_files = set(os.listdir(self.folder_path))

            new_files = current_files - set(self.file_info.keys())
            for new_file in new_files:
                print(f"{new_file} - New File")

            deleted_files = set(self.file_info.keys()) - current_files
            for deleted_file in deleted_files:
                print(f"{deleted_file} - Deleted")

            print(f"Snapshot Time: {self.snapshot_time}")
            for filename, info in self.file_info.items():
                Instance = self.get_file_instance(filename)
                if Instance:
                    current_info = Instance(filename).get_info(self.folder_path)
                    if current_info:
                        if current_info != info:
                            print(f"{filename} - Changed")
                        else:
                            print(f"{filename} - No Change")
                    else:
                        print(f"{filename} - Not found")

def scheduled_detection(monitor):
    while True:
        monitor.monitor_folder()
        monitor.commit()  
        monitor.status()
        time.sleep(5)

if __name__ == "__main__":
    folder_path = r'C:\Users\iulia\OneDrive\Desktop\CosieruCatalin_oop\DorogoiIulian_Lab3_oop'
    monitor = FileMonitor(folder_path)

    detection_thread = threading.Thread(target=scheduled_detection, args=(monitor,))
    detection_thread.start()

    while True:
        command = input(
            "Enter command (info <filename>/status/exit): ").strip().split()
        if command[0] == 'info' and len(command) == 2:
            monitor.info(command[1])
        elif command[0] == 'status':
            monitor.status()
        elif command[0] == 'exit':
            break
        else:
            print("Invalid command.")

