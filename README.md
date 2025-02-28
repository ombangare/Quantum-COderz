<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>QR Code Attendance</title>
</head>
<body>
    <h2>Scan QR Code to Mark Attendance</h2>
    
    <video id="preview"></video>
    
    <input type="text" id="student_id" placeholder="Enter Student ID">
    <button onclick="submitAttendance()">Submit Attendance</button>
    
    <script src="https://rawgit.com/schmich/instascan-builds/master/instascan.min.js"></script>
    <script>
        let scanner = new Instascan.Scanner({ video: document.getElementById('preview') });

        scanner.addListener('scan', function (content) {
            let studentId = document.getElementById("student_id").value;
            if (!studentId) {
                alert("Enter Student ID before scanning!");
                return;
            }
            let url = `${content}&student_id=${studentId}`;
            fetch(url).then(response => response.json()).then(data => alert(data.message));
        });

        Instascan.Camera.getCameras().then(function (cameras) {
            if (cameras.length > 0) {
                scanner.start(cameras[0]);
            } else {
                console.error("No cameras found.");
            }
        }).catch(function (e) {
            console.error(e);
        });

        function submitAttendance() {
            alert("Please scan the QR code.");
        }
    </script>
</body>
</html>
from flask import Flask, request, jsonify, render_template
import qrcode
import gspread
from oauth2client.service_account import ServiceAccountCredentials
import os

app = Flask(__name__)

# Google Sheets API Setup
scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
creds = ServiceAccountCredentials.from_json_keyfile_name("credentials.json", scope)
client = gspread.authorize(creds)
sheet = client.open("Attendance").sheet1  # Replace with your Google Sheet name

# Create 'static' folder if not exists
if not os.path.exists("static"):
    os.makedirs("static")

@app.route("/")
def home():
    return render_template("index.html")  # Frontend QR scanner

@app.route("/generate_qr/<lecture_id>")
def generate_qr(lecture_id):
    """ Generates a QR code for the lecture """
    qr_data = f"http://localhost:5000/mark_attendance?lecture_id={lecture_id}"
    qr = qrcode.make(qr_data)
    qr_path = f"static/lecture_{lecture_id}.png"
    qr.save(qr_path)
    return f"<h3>QR Code Generated for Lecture {lecture_id}</h3><img src='/{qr_path}' width='300px'>"

@app.route("/mark_attendance", methods=["GET"])
def mark_attendance():
    """ Logs attendance when student scans QR code """
    student_id = request.args.get("student_id")
    lecture_id = request.args.get("lecture_id")

    if not student_id or not lecture_id:
        return jsonify({"error": "Missing parameters"}), 400

    sheet.append_row([student_id, lecture_id])
    return jsonify({"message": "Attendance Recorded Successfully!"})

if __name__ == "__main__":
    app.run(debug=True)
