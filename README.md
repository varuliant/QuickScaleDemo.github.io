# QuickScaleDemo.github.io
Tools for obtained a daily F-Layer critical frequency values determined by a Frequency-Time Intensity (FTI) Plot
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>A Software Cemonstration for F layer critical frequency using Quick Scale method from FTI plots</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            padding: 0;
            background-color: #f4f4f9;
        }

        h2 {
            color: #2c3e50;
            text-align: center;
        }

        .container {
            max-width: 900px;
            margin: 0 auto;
            background-color: white;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }

        p {
            font-size: 14px;
            line-height: 1.6;
        }

        .instructions p {
            margin-left: 20px;
        }

        .button-group {
            display: flex;
            flex-wrap: wrap;
            justify-content: center;
            gap: 10px;
            margin-top: 20px;
        }

        button {
            background-color: #3498db;
            color: white;
            border: none;
            padding: 10px 15px;
            font-size: 14px;
            cursor: pointer;
            border-radius: 5px;
            transition: background-color 0.3s ease;
        }

        button:hover {
            background-color: #2980b9;
        }

        canvas {
            display: block;
            margin: 20px auto;
            border: 1px solid #ccc;
            cursor: crosshair;
        }

        input[type="file"] {
            display: block;
            margin: 20px auto;
        }

        .selected-values {
            text-align: center;
            margin-top: 15px;
        }

        .selected-values span {
            font-weight: bold;
        }
    </style>
</head>
<body>
    <div class="container">
        <h2>Quick scale foF2 from FTI image</h2>

        <p>The Quickscale foF2 tool was developed by Varuliantor Dear from the Space Research Center, BRIN Indonesia in January 2025.</p>

        <div class="instructions">
            <p>To use this tool, the steps are follows:</p>
            	<p>(i) Upload a FTI image, </p>
            	<p>(ii) Press the "Left/Right-X", "Left/Right-Y" button, and select the X and Y axis points in the image, </p>
            	<p>(iii) Select the "Image Cal" button, and type the range of real value, </p>
            	<p>(iv) Make a foF2 trace by selecting the "foF2 Plot", and </p>
            	<p>(v) Download the result as CSV file format</p>
            <p>The calibration process is performed only <b> once if the selected Image has the same dimensions </b> as the previous one. The plot trace result will be deleted when selecting a new Image. After the plot trace is complete, please press the "Download CSV" button to get a file containing the foF2 parameter values ​​consisting of 4 columns, namely: <b> Parameter, Segment, Decimal Clock, and Frequency </b> in MHz units.</p>

        </div>

        <input type="file" id="imageInput" accept="image/*">
        <p id="imageDimensions"></p>

        <canvas id="canvas"></canvas>

        <div class="selected-values">
            <p>Selected points: Left-X: <span id="selectedMinX"></span>, Right-X: <span id="selectedMaxX"></span>, Upper-Y: <span id="selectedMinY"></span>, Bottom-Y: <span id="selectedMaxY"></span></p>
        </div>

        <div class="button-group">
            <button onclick="selectMinX()">Left-X</button>
            <button onclick="selectMaxX()">Right-X</button>
            <button onclick="selectMinY()">Upper-Y</button>
            <button onclick="selectMaxY()">Bottom-Y</button>
            <button onclick="scaleImage()">Image Cal</button>
        </div>
        <div class="button-group">
            <button onclick="clearDrawing()">Erase Trace</button>
            <button onclick="setMode('foF2')">foF2 Plot</button>
            <button onclick="downloadCSV()">Download CSV</button>
        </div>
    </div>

 <script>
        let canvas = document.getElementById("canvas");
        let ctx = canvas.getContext("2d");
        let drawing = false;
        let mode = "foF2";
        let foF2_points = [];
        let img = new Image();
        let scalingPoints = { minX: null, maxX: null, minY: null, maxY: null };
        let selecting = "";
        let minXReal, maxXReal, minYReal, maxYReal;
        let uploadedFileName = ""; // To store the uploaded file name

        document.getElementById("imageInput").addEventListener("change", function(event) {
            let file = event.target.files[0];
            if (file) {
                let reader = new FileReader();
                reader.onload = function(e) {
                    img.src = e.target.result;
                    // Store the file name without extension
                    uploadedFileName = file.name.replace(/\.[^/.]+$/, ""); 
                };
                reader.readAsDataURL(file);
            }
        });

        img.onload = function() {
            canvas.width = img.width;
            canvas.height = img.height;
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.drawImage(img, 0, 0);
            document.getElementById("imageDimensions").innerText = `Image Dimensions: ${img.width} x ${img.height}`;
            clearDrawing();
        };

        canvas.addEventListener("mousedown", function(event) {
            let rect = canvas.getBoundingClientRect();
            let x = event.clientX - rect.left;
            let y = event.clientY - rect.top;

            if (selecting) {
                scalingPoints[selecting] = { x, y };
                document.getElementById(`selected${selecting.charAt(0).toUpperCase() + selecting.slice(1)}`).innerText = `(${x}, ${y})`;
                selecting = "";
                return;
            }
            drawing = true;
            if (mode === "foF2") foF2_points.push([{x, y}]);  // Start a new line for foF2
        });
        
        canvas.addEventListener("mouseup", () => drawing = false);
        canvas.addEventListener("mouseleave", () => drawing = false);

        canvas.addEventListener("mousemove", function(event) {
            if (!drawing) return;
            let rect = canvas.getBoundingClientRect();
            let x = event.clientX - rect.left;
            let y = event.clientY - rect.top;
            if (mode === "foF2") foF2_points[foF2_points.length - 1].push({x, y});  // Add new point to the last line
            draw();
        });

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.drawImage(img, 0, 0);
            drawLines(foF2_points, "red");
            drawLines(fmin_points, "white");
        }

        function drawLines(pointsArray, color) {
            ctx.beginPath();
            ctx.strokeStyle = color;
            ctx.lineWidth = 4;
            pointsArray.forEach(points => {
                points.forEach((p, index) => {
                    if (index === 0) ctx.moveTo(p.x, p.y);
                    else ctx.lineTo(p.x, p.y);
                });
            });
            ctx.stroke();
        }

        function selectMinX() { selecting = "minX"; }
        function selectMaxX() { selecting = "maxX"; }
        function selectMinY() { selecting = "minY"; }
        function selectMaxY() { selecting = "maxY"; }

        function scaleImage() {
            if (!scalingPoints.minX || !scalingPoints.maxX || !scalingPoints.minY || !scalingPoints.maxY) {
                alert("Please select all scaling points first!");
                return;
            }
            
            minXReal = parseFloat(prompt("Enter real Min X value:"));
            maxXReal = parseFloat(prompt("Enter real Max X value:"));
            minYReal = parseFloat(prompt("Enter real Min Y value:"));
            maxYReal = parseFloat(prompt("Enter real Max Y value:"));
        }

        function clearDrawing() {
            foF2_points = [];
            fmin_points = [];
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.drawImage(img, 0, 0);
        }

        function setMode(selectedMode) {
            mode = selectedMode;
        }

function downloadCSV() {
    let csvContent = "data:text/csv;charset=utf-8,Parameter,Segment,Dec_Clock,Frequency\n";
    
    // Fungsi untuk mengkonversi titik ke nilai real
    function convertPoint(p) {
        return {
            x: minXReal + ((p.x - scalingPoints.minX.x) / (scalingPoints.maxX.x - scalingPoints.minX.x)) * (maxXReal - minXReal),
            y: minYReal + ((scalingPoints.maxY.y - p.y) / (scalingPoints.maxY.y - scalingPoints.minY.y)) * (maxYReal - minYReal)
        };
    }

    // Fungsi untuk mengonversi dan menambahkan data ke dalam CSV
    function addDataToCSV(points, label) {
        points.forEach((line, index) => {
            line.forEach(p => {
                let realP = convertPoint(p);
                let jamDec = (index + 1) * 0.1;  // Anggap setiap garis sebagai interval jam
                csvContent += `${label},${jamDec},${realP.x},${realP.y}\n`;
            });
        });
    }

    // Tambahkan data dari foF2 (garis merah)
    addDataToCSV(foF2_points, "foF2");


    // Encode CSV content and create a link for download
    let encodedUri = encodeURI(csvContent);
    let link = document.createElement("a");
    let csvFileName = uploadedFileName ? uploadedFileName + "_data.csv" : "data.csv";
    link.setAttribute("href", encodedUri);
    link.setAttribute("download", csvFileName);
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
}
    </script>
</body>
</html>
