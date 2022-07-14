<html>

<head>
    <title>reading file</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/2.9.4/Chart.js"></script>

    <script type="text/javascript">

        let ElmDump = "";     // global string containing the ELM data from the Clarity (or loaded from a log file)
        let ModDate = "";     // global string containing the last modified date

        function hex2ascii(str1)                       // Input an array of ascii/hex bytes.  Convert these to ASCII and output a string
        {
            var str = '';
            if (typeof str1 === "undefined") {         // If undefined (Legacy log file), then return a zero
            }
            else {
                for (var n = 0; n < str1.length; n++) {
                    str += String.fromCharCode(parseInt(str1[n], 16));
                }
            }
            return str;
        }

        function PullInteger(str1, iPos, nBytes)       // Input an array of ascii/hex bytes.  Form a multibyte integer value starting at position iPos
        {
            var iValue = 0;
            if (typeof str1 === "undefined") {         // If undefined (Legacy log file), then return a zero
            }
            else {
                for (var n = 0; n < nBytes; n++) {
                    iValue += 2 ** (8 * (nBytes - n - 1)) * parseInt(str1[iPos + n], 16);
                }
            }
            return iValue;
        }


        function download(content, fileName, contentType) {         // Allows you to download a string (content) to 'fileName' through dialog box 
            var a = document.createElement("a");                    // contentType can be 'text/plain'
            var file = new Blob([content], { type: contentType });
            a.href = URL.createObjectURL(file);
            a.download = fileName;
            a.click();
        }


        function SaveIt() {
            var CurrentTime = new Date();
            // Build a filename of the form 'SessJava_yyyy_mmm_dd_hh_mm_ss.txt'
            fName = "SessJava_" + CurrentTime.toDateString().substring(11, 15) + "_" + CurrentTime.toDateString().substring(4, 7) + "_" + CurrentTime.toDateString().substring(8, 10)
            fName += "_" + CurrentTime.toTimeString().substring(0, 8).replaceAll(":", "_") + ".txt"
            download(ElmDump, fName, 'text/plain');
        }


        function readELM(that) {                                 // Browse to an existing log file, and load it onto 'ElmDump'
            if (that.files && that.files[0]) {
                var reader = new FileReader();
                reader.onload = function (e) {
                    console.log(e.target.result);
                    ElmDump = e.target.result;
                    ModDate = that.files[0].lastModifiedDate;    // This is the Last Modified date of the input file
                };
                reader.readAsText(that.files[0]);
            }
        }


        function ClarityCollect() {                              // Read data from Clarity, Save it in a log file
            console.log("Accessing OBD Data through ELM327")
            QueryELM();
            ModDate = new Date() ;
            console.log("Done with it !")
        }

        async function QueryELM() {

            // Example from: https://stackoverflow.com/questions/63456661/beginner-problem-with-chrome-navigator-serial
            // And here:  https://web.dev/serial/

            const ElmCmds = ["ATZ",
                "ATI",
                "ATE1",
                "ATL0",
                "ATSP7",
                "ATH1",
                "ATS0",
                "ATSHDB33F1",
                "0902",
                "ATSHDA60F1",
                "227022",
                "227060",
                "ATSHDA01F1",
                "222028",
                "22202A",
                "22202C",
                "ATSHDA16F1",
                "222028",
                "22202A",
                "22202C",
                "ATSHDA15F1",
                "222021",
                "222029",
                "ATSHDA28F1",
                "224004",
                "224005",
                "ATSHDA2AF1",
                "224201",
                "ATSHDA0EF1",
                "222240",
                "222613"];

            var CR = String.fromCharCode(13);
            var LogString = "";
            var buf = "";

            try {
                // Prompt user to select any serial port.
                const port = await navigator.serial.requestPort();
                // Wait for the serial port to open.
                await port.open({ baudRate: 115200 });

                const textDecoder = new TextDecoderStream();

                const readableStreamClosed = port.readable.pipeTo(textDecoder.writable);
                const reader = textDecoder.readable.getReader();
                const textEncoder = new TextEncoderStream();
                const writableStreamClosed = textEncoder.readable.pipeTo(port.writable);
                const writer = textEncoder.writable.getWriter();

                for (var iCmd = 0; iCmd < ElmCmds.length; iCmd++) {
                    console.log("Sending Command " + iCmd + " " + ElmCmds[iCmd]);
                    buf = "";
                    writer.write(ElmCmds[iCmd] + CR);   // Send command to the ELM

                    while (!buf.includes(">")) {        // Read until ELM provides a prompt
                        const { value, done } = await reader.read();
                        buf += value;
                    }

                    console.log(buf);
                    LogString += buf;
                }

                // Close the serial ports
                reader.cancel();
                await readableStreamClosed.catch(() => { /* Ignore the error */ });
                writer.close();
                await writableStreamClosed;
                await port.close();

                ElmDump = LogString;
                SaveIt();                                // Put the ELM Query into a time-stamped file

            }
            catch (err) {
                alert(err);
            }

        }  // End of function QueryELM()



        function ReportGen() {                       // Generate the report
            console.log("Generating the Report");
            console.log(ElmDump);
            // Comments about PID Arrays for Clarity
            // Valid Headers & PID's:
            //    HDR           PID    LEN    NAME
            //  DB33F1           902    14    A0E_902
            // (aka DA0EF1)   222240    36    A0E_222240
            //     "          222613    54    A0E_222613
            //  DA60F1        227022    56    A60_227022
            //     "          227060    56    A60_227060
            //  DA01F1        222028   245    A01_222028
            //     "          22202A   245    A01_22202A
            //     "          22202C   245    A01_22202C
            //  DA16F1        222028   245    A16_222028
            //     "          22202A   245    A16_22202A
            //     "          22202C   245    A16_22202C
            //  DA15F1        222021   245    A15_222021
            //     "          222029   245    A15_222029
            //
            //  DA28F1        224004    56    A28_224004 Below this, not in original ClarityELM
            //     "          224005    56    A28_224005
            //
            //  DA2AF1        224201    56    A2A_224201

            var fileContentArray = ElmDump.split(/\r|\n/);
            var pids = fileContentArray.filter(i => i.startsWith("18D"));
            console.log(pids);   // Log all of the PID readings
            var Data, FrameCnt, FrameCnt_exp, Bcnt, Hdr, Len;

            for (var line = 0; line < pids.length - 1; line++) {
                console.log("Line # " + line + " Frame Type = 0x" + pids[line].substring(8, 10));
                if (pids[line].substring(8, 10) == "10") {                                          // This is the start of a message
                    Data = [];
                    FrameCnt_exp = 0x10;
                    Bcnt = 0;                                                                       // Count the Bytes in a message

                    Hdr = pids[line].substring(3, 4) + pids[line].substring(6, 8)                   // This is the Header
                    PID = (parseInt(pids[line].substring(12, 18), 16) & 0x3FFFFF).toString(16)      // This is the PID number
                    Len = parseInt(pids[line].substring(10, 12), 16)                                // How many bytes in the message
                    console.log("Processing a Frame (0x10) " + "Hdr = " + Hdr + " PID= " + PID + " Len= " + Len);
                    for (Bcnt = 0; Bcnt < Len; Bcnt++) {                                            // Read in all the message bytes
                        if ((Bcnt + 1) % 7 == 0) {                                                  // Get a new line if needed
                            line++;
                            if (FrameCnt_exp == 16) { FrameCnt_exp = 32 }                           // Account for initial frame (0x10)
                            FrameCnt_exp++;                                                         // Increment the Frame Count, Modulo it 
                            if (FrameCnt_exp == 48) { FrameCnt_exp = 32 }
                        }
                        FrameCnt = parseInt(pids[line].substring(8, 10), 16)                        // Starts at 0x21, then 0x22, .. 0x2F, 0x20
                        while (FrameCnt == 3) {                                                     // This while skips any '03' frames whicn may be mixed in
                            line++;
                            FrameCnt = parseInt(pids[line].substring(8, 10), 16)
                        }
                        if (FrameCnt != FrameCnt_exp) {
                            console.log("Hdr = " + Hdr + " PID= " + PID + "***** Frame Counter Error **** Read=" + FrameCnt + " Expected=" + FrameCnt_exp);
                        }
                        Data.push(pids[line].substring(10 + 2 * ((Bcnt + 1) % 7), 10 + 2 * ((Bcnt + 1) % 7) + 2));
                    }
                    console.log("Done with a Message");

                    switch (Hdr) {                                              // Populate each of the data arrays

                        case "A0E":
                            switch (PID) {
                                case "90201":
                                    var B33_902 = Data;
                                    break;
                                case "222240":
                                    var A0E_222240 = Data;
                                    break;
                                case "222613":
                                    var A0E_222613 = Data;
                                    break;
                            }
                            break;

                        case "A60":
                            switch (PID) {
                                case "227022":
                                    var A60_227022 = Data;
                                    break;
                                case "227060":
                                    var A60_227060 = Data;
                                    break;
                            }
                            break;

                        case "A01":
                            switch (PID) {
                                case "222028":
                                    var A01_222028 = Data;
                                    break;
                                case "22202a":
                                    var A01_22202A = Data;
                                    break;
                                case "22202c":
                                    var A01_22202C = Data;
                                    break;
                            }
                            break;

                        case "A16":
                            switch (PID) {
                                case "222028":
                                    var A16_222028 = Data;
                                    break;
                                case "22202a":
                                    var A16_22202A = Data;
                                    break;
                                case "22202c":
                                    var A16_22202C = Data;
                                    break;
                            }
                            break;

                        case "A15":
                            switch (PID) {
                                case "222021":
                                    var A15_222021 = Data;
                                    break;
                                case "222029":
                                    var A15_222029 = Data;
                                    break;
                            }
                            break;

                        case "A28":
                            switch (PID) {
                                case "224004":
                                    var A28_224004 = Data;
                                    break;
                                case "224005":
                                    var A28_224005 = Data;
                                    break;
                            }
                            break;

                        case "A2A":
                            switch (PID) {
                                case "224201":
                                    var A2A_224201 = Data;
                                    break;
                            }
                            break;
                    }

                }
            }

            // Create the Report Here
            //

            var AllPrnt = "", CellPrint="";

            AllPrnt += "Electric Powertrain Report " + ModDate + "<br />";    // Use 'new Date()' for current date
            AllPrnt += "VIN: " + hex2ascii(B33_902).substring(3) + "<br />";

            var Odo = PullInteger(A60_227022, 9, 3);
            AllPrnt += "Odometer: " + Odo + " mi.<br />";
            AllPrnt += "Distance traveled since Battery Connected: " + (0.621371 * PullInteger(A0E_222240, 13, 4)).toFixed(0) + " mi.<br />"; // converted km to mi.
            AllPrnt += "Distance since DTC Cleared: " + (0.621371 * PullInteger(A01_22202A, 38, 4)).toFixed(0) + " mi.<br /> <br />"; // converted km to mi.

            var BTemp1A = PullInteger(A01_22202C, 62, 2) / 10.0;
            var BTemp2A = PullInteger(A01_22202C, 64, 2) / 10.0;
            var BTemp3A = PullInteger(A01_22202C, 66, 2) / 10.0;
            var BTemp1B = PullInteger(A16_22202C, 62, 2) / 10.0;
            var BTemp2B = PullInteger(A16_22202C, 64, 2) / 10.0;
            var BTemp3B = PullInteger(A16_22202C, 66, 2) / 10.0;
            var BTemp4B = PullInteger(A16_22202C, 68, 2) / 10.0;
            AllPrnt += "HV Battery Module 1A Temperature: " + BTemp1A.toFixed(1) + "C,  " + (9.0 / 5.0 * BTemp1A + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "HV Battery Module 2A Temperature: " + BTemp2A.toFixed(1) + "C,  " + (9.0 / 5.0 * BTemp2A + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "HV Battery Module 3A Temperature: " + BTemp3A.toFixed(1) + "C,  " + (9.0 / 5.0 * BTemp3A + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "HV Battery Module 1B Temperature: " + BTemp1B.toFixed(1) + "C,  " + (9.0 / 5.0 * BTemp1B + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "HV Battery Module 2B Temperature: " + BTemp2B.toFixed(1) + "C,  " + (9.0 / 5.0 * BTemp2B + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "HV Battery Module 3B Temperature: " + BTemp3B.toFixed(1) + "C,  " + (9.0 / 5.0 * BTemp3B + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "HV Battery Module 4B Temperature: " + BTemp4B.toFixed(1) + "C,  " + (9.0 / 5.0 * BTemp4B + 32.0).toFixed(1) + "F <br />";

            var ECTemp1 = PullInteger(A01_22202C, 204, 2) / 10.0;
            var ECTemp2 = PullInteger(A01_22202C, 206, 2) / 10.0;
            var ECTemp3 = PullInteger(A01_22202C, 208, 2) / 10.0;
            var ECTemp4 = PullInteger(A01_22202C, 210, 2) / 10.0;
            AllPrnt += "ES Coolant Temperature 1: " + ECTemp1.toFixed(1) + "C,  " + (9.0 / 5.0 * ECTemp1 + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "ES Coolant Temperature 2: " + ECTemp2.toFixed(1) + "C,  " + (9.0 / 5.0 * ECTemp2 + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "ES Coolant Temperature 3: " + ECTemp3.toFixed(1) + "C,  " + (9.0 / 5.0 * ECTemp3 + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "ES Coolant Temperature 4: " + ECTemp4.toFixed(1) + "C,  " + (9.0 / 5.0 * ECTemp4 + 32.0).toFixed(1) + "F <br /><br />";

            var ATempIn = PullInteger(A01_22202A, 107, 1)
            var ATempOut = PullInteger(A01_22202A, 108, 1)
            AllPrnt += "Air Temperature Inside Vehicle: " + ATempIn.toFixed(1) + "C,  " + (9.0 / 5.0 * ATempIn + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "Air Temperature Outside Vehicle: " + ATempOut.toFixed(1) + "C,  " + (9.0 / 5.0 * ATempOut + 32.0).toFixed(1) + "F <br />";
            AllPrnt += "A/C Freon Pressure:  " + (0.145038 * PullInteger(A0E_222613, 18, 2)).toFixed(1) + " PSI <br /><br />";   // Converted from kPa to PSI

            AllPrnt += "HV Battery Voltage A: " + PullInteger(A01_22202A, 64, 2) / 20.0 + "V<br />";
            AllPrnt += "HV Cell Max SOC A: " + PullInteger(A01_22202A, 76, 2) / 100.0 + "%<br />";
            AllPrnt += "HV Cell Min SOC A: " + PullInteger(A01_22202A, 78, 2) / 100.0 + "%<br />";
            AllPrnt += "SOC: " + PullInteger(A01_22202A, 53, 1) + "%<br /> <br />";

            AllPrnt += "HV Battery Voltage B: " + PullInteger(A16_22202A, 64, 2) / 20.0 + "V<br />";
            AllPrnt += "HV Cell Max SOC B: " + PullInteger(A16_22202A, 76, 2) / 100.0 + "%<br />";
            AllPrnt += "HV Cell Min SOC B: " + PullInteger(A16_22202A, 78, 2) / 100.0 + "%<br /><br />";

            var BCapTot = PullInteger(A01_22202A, 170, 2) / 100.0;
            var BCapB = PullInteger(A16_22202A, 170, 2) / 100.0;
            AllPrnt += "HV Battery Capacity (A+B): " + BCapTot.toFixed(2) + "ah<br />";
            AllPrnt += "HV Battery Capacity A: " + (BCapTot - BCapB).toFixed(2) + "ah<br />";
            AllPrnt += "HV Battery Capacity B: " + BCapB.toFixed(2) + "ah<br /><br />";

            AllPrnt += "Input Voltage of Normal Charger: " + (PullInteger(A15_222021, 33, 2) / 10.0).toFixed(1) + "V<br />";
            AllPrnt += "Output Voltage of Normal Charger: " + (PullInteger(A15_222021, 37, 2) / 10.0).toFixed(1) + "V<br />";
            AllPrnt += "Charging Voltage Target: " + (PullInteger(A15_222029, 37, 2) / 5.0).toFixed(1) + "mV<br />";
            AllPrnt += "Current Limit during Plug-in Charging: -" + ((65536 - PullInteger(A15_222029, 54, 2)) / 10.0).toFixed(1) + "A<br /><br />";  // Make it negative

            AllPrnt += "<b>Maintenance Minder</b><br />";
            AllPrnt += "A - Oil & Filter        : " + PullInteger(A60_227060, 17, 2) + " days<br />";
            AllPrnt += "0 - General Inspection  : " + PullInteger(A60_227060, 29, 2) + " days<br />";
            AllPrnt += "1 - Rotate Tires        : " + PullInteger(A60_227060, 31, 2) + " days<br />";
            AllPrnt += "2 - Cabin Filter        : " + PullInteger(A60_227060, 33, 2) + " days<br />";
            AllPrnt += "3 - Transmission Fluid  : " + PullInteger(A60_227060, 35, 2) + " days<br />";
            AllPrnt += "4 - Spark Plugs & Valves: " + PullInteger(A60_227060, 19, 2) + " days<br />";
            AllPrnt += "5 - Engine Coolant      : " + PullInteger(A60_227060, 39, 2) + " days<br />";
            AllPrnt += "7 - Brake Fluid         : " + PullInteger(A60_227060, 43, 2) + " days<br />";
            AllPrnt += "8 - Air Filter          : " + PullInteger(A60_227060, 45, 2) + " days<br /><br />";

            // Individual Cell Voltages
            // Bank A
            CellPrint += "<br />" + "<b>A Bank Cell Voltages, Cells 01-84 (mV):</b>" + "<br />" +"<br />"+ "01 "
            var ACells = [];
            var Amin = 9999, Amax = 0, Aavg = 0;
            for (var n = 0; n < 84; n++) {
                ACells.push(PullInteger(A01_222028, 30 + 2 * n, 2) / 5.0);
              
                CellPrint += ACells[n].toFixed(1) + " " ;
                if ((n+1) % 10 == 0) {
                    CellPrint += "<br />" + (n+2).toFixed(0) + " " ;
                }

                Amin = Math.min(Amin, ACells[n]);
                Amax = Math.max(Amax, ACells[n]);
                Aavg += ACells[n];
            }
            Aavg = Aavg / 84.0;

            // Bank B
            CellPrint += "<br />" + "<br />" + "<br />" + "<b>B Bank Cell Voltages, Cells 01-84 (mV):</b>" + "<br />" +"<br />"+ "01 "
            var BCells = [];
            var Bmin = 9999, Bmax = 0, Bavg = 0, iCell = [];
            for (var n = 0; n < 84; n++) {
                BCells.push(PullInteger(A16_222028, 30 + 2 * n, 2) / 5.0);
                iCell.push(n);                                         // Index for X-axis

                CellPrint += BCells[n].toFixed(1) + " " ;
                if ((n+1) % 10 == 0) {
                    CellPrint += "<br />" + (n+2).toFixed(0) + " " ;
                }

                Bmin = Math.min(Bmin, BCells[n]);
                Bmax = Math.max(Bmax, BCells[n]);
                Bavg += BCells[n];
            }
            var MaxMax = 0, MinMin = 0 ;
            Bavg = Bavg / 84.0;
            MaxMax = Math.max(Amax, Bmax);
            MinMin = Math.min(Amin, Bmin);

            // Show the Cell Statistics
            AllPrnt += '<table style="width:60%">';
            AllPrnt += "<tr><td><td></tr>";
            AllPrnt += "<tr><td><b>Cell Statistics, mV:</b><td></tr>";
            AllPrnt += "<tr><td></td><td>min</td><td>max</td><td>delta</td><td>avg</td></tr>";
            AllPrnt += "<tr><td>Bank A</td>" + "<td>" + Amin.toFixed(1) + "</td>" + "<td>" + Amax.toFixed(1) + "</td>" + "<td>" + (Amax - Amin).toFixed(2) + "</td>" + "<td>" + Aavg.toFixed(1) + "</td></tr>";
            AllPrnt += "<tr><td>Bank B</td>" + "<td>" + Bmin.toFixed(1) + "</td>" + "<td>" + Bmax.toFixed(1) + "</td>" + "<td>" + (Bmax - Bmin).toFixed(2) + "</td>" + "<td>" + Bavg.toFixed(1) + "</td></tr>";
            AllPrnt += "<tr><td><td></tr>";


            var ABAvg = Math.round((Aavg + Bavg) / 2.0);   // Average both banks, and round to nearest mV

            // Create a plot for the Cell Voltages
            var myChart = new Chart("myChart", {           // Initial Settings for Chart
                type: "line",
                data: {
                    labels: iCell,
                    datasets: [{
                        label: "A Bank",
                        data: ACells,
                        borderColor: "red",
                        fill: false
                    }, {
                        label: "B Bank",
                        data: BCells,
                        borderColor: "green",
                        fill: false
                    }]
                },
                options: {
                    title: {
                        display: true,
                        text: 'Clarity Cell Voltages'
                    },

                    scales: {
                        xAxes: [{
                            scaleLabel: {
                                display: true,
                                labelString: 'Cell Number'
                            },
                        }],
                        yAxes: [{
                            scaleLabel: {
                                display: true,
                                labelString: 'Cell Voltage (mV)'
                            },
                            ticks: {
                                steps: 10,
                                stepValue: 1,
                                min: 0,
                                max: 4000
                            }
                        }]
                    }
                }

            });

            //change the scale in accordance with the data
            myChart.config.options.scales.yAxes[0].ticks.min = Math.round((MaxMax+2.5)/5.0)*5 ;         // Truncate Max to the nearest 5
            myChart.config.options.scales.yAxes[0].ticks.max = Math.round((MinMin+2.5)/5.0)*5 - 5 ;     // Truncate Min to the nearest 5
            myChart.update();

            document.getElementById('main').innerHTML = AllPrnt;             // Send the report to the browser page
            document.getElementById('cells').innerHTML = CellPrint;          // Send Cell Voltages to the browser page

        }  // End of function ReportGen()


    </script>
</head>

<body>
    <input type="button" onclick='ClarityCollect()' value="Load Data from Clarity">
    &nbsp; Or, &nbsp; Read Existing Log File:
    <input type="file" onchange='readELM(this)'>
    ==>
    <input type="button" onclick='ReportGen()' value="Generate Report">
    <div id="main"></div>
    <canvas id="myChart" style="width:100%;max-width:800px"></canvas>
    <div id="cells"></div>
</body>

</html>
