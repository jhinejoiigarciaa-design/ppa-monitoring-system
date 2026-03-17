<!DOCTYPE html>
<html>
<head>
<title>PPA Monitoring System V4</title>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<style>

body{
font-family:Arial;
background:#eef2f7;
margin:20px;
}

h1{text-align:center;}

.dashboard{
display:flex;
gap:15px;
margin-bottom:20px;
}

.card{
flex:1;
padding:20px;
color:white;
border-radius:8px;
text-align:center;
}

.total{background:#34495e;}
.done{background:#27ae60;}
.ongoing{background:#f39c12;}
.pipeline{background:#e74c3c;}
.forimpl{background:#191970;}

table{
width:100%;
border-collapse:collapse;
background:white;
}

th,td{
border:1px solid #ddd;
padding:8px;
text-align:center;
}

th{
background:#2c3e50;
color:white;
}

input,select,button{
padding:6px;
margin:5px;
}

button{
background:#2c3e50;
color:white;
border:none;
cursor:pointer;
}

button:hover{
background:#1a252f;
}

.formbox{
background:white;
padding:15px;
border-radius:8px;
margin-bottom:20px;
}

#loginPage{
width:300px;
margin:100px auto;
background:white;
padding:20px;
border-radius:10px;
text-align:center;
}

</style>
</head>

<body>

<!-- LOGIN -->
<div id="loginPage">
<h2>PPA System Login</h2>
<input id="username" placeholder="Username"><br>
<input id="password" type="password" placeholder="Password"><br>
<button onclick="login()">Login</button>
<p id="loginMsg" style="color:red;"></p>
</div>

<!-- MAIN SYSTEM -->
<div id="mainSystem" style="display:none;">

<h1>PPA IMPLEMENTATION MONITORING</h1>

<div class="dashboard">
<div class="card total">TOTAL<h2 id="total">0</h2></div>
<div class="card done">DONE<h2 id="completed">0</h2></div>
<div class="card ongoing">ONGOING<h2 id="ongoing">0</h2></div>
<div class="card pipeline">PIPELINE<h2 id="pipelineCount">0</h2></div>
<div class="card forimpl">FOR IMPLEMENTATION<h2 id="forImpl">0</h2></div>
<div class="card" style="background:#c0392b;">DELAYED<h2 id="delayed">0</h2></div>
</div>

<div class="formbox">
<h3>Add New PPA</h3>

<input id="code" placeholder="AIP Code">
<input id="desc" placeholder="Description">
<input id="budget" type="number" placeholder="Budget">

<select id="time">
<option>Year Round</option>
<option>On Time</option>
<option>Early</option>
<option>Delayed</option>
<option>Not Implemented</option>
</select>

<select id="status">
<option>For Implementation</option>
<option>Pipeline</option>
<option>Ongoing</option>
<option>Completed</option>
<option>Dropped</option>
</select>

<select id="fund">
<option>Obligated</option>
<option>Disbursed</option>
<option>Liquidated</option>
<option>Utilized</option>
<option>Unutilized</option>
</select>

<button onclick="addPPA()">Add</button>
</div>

<table id="ppaTable">
<tr>
<th>Code</th>
<th>Description</th>
<th>Budget</th>
<th>Timeliness</th>
<th>Status</th>
<th>Fund</th>
<th>Action</th>
</tr>
</table>

<canvas id="chart" height="100"></canvas>

</div>

<script>

// 🔗 GOOGLE SHEETS API (PUT YOUR URL HERE)
const API_URL = "https://script.google.com/macros/s/AKfycbxaplYZr1VdMcSgKBWBrRdGgk0D9HutzXE-v27YMHab1kivsiKcnlXQn_yRO2Gfbmb4ig/exec";

// 🔐 LOGIN
function login(){
let user=document.getElementById("username").value;
let pass=document.getElementById("password").value;

if(user=="admin" && pass=="admin123"){
document.getElementById("loginPage").style.display="none";
document.getElementById("mainSystem").style.display="block";
}else{
document.getElementById("loginMsg").innerText="Invalid login!";
}
}

// ➕ ADD PPA
function addPPA(){

let code=codeInput("code");
let desc=codeInput("desc");
let budget=codeInput("budget");
let time=codeInput("time");
let status=codeInput("status");
let fund=codeInput("fund");

let table=document.getElementById("ppaTable");
let row=table.insertRow();

row.innerHTML=`
<td>${code}</td>
<td>${desc}</td>
<td>${budget}</td>
<td>${time}</td>
<td>${status}</td>
<td>${fund}</td>
<td>
<button onclick="editRow(this)">Edit</button>
<button onclick="deleteRow(this)">Delete</button>
</td>
`;

saveToSheet(code,desc,budget,time,status,fund);

monitor();
}

// 🔄 HELPER
function codeInput(id){
return document.getElementById(id).value;
}

// 🗑 DELETE
function deleteRow(btn){
btn.parentElement.parentElement.remove();
monitor();
}

// ✏ EDIT
function editRow(btn){
let row=btn.parentElement.parentElement;
let cells=row.cells;

document.getElementById("code").value=cells[0].innerText;
document.getElementById("desc").value=cells[1].innerText;
document.getElementById("budget").value=cells[2].innerText;
document.getElementById("time").value=cells[3].innerText;
document.getElementById("status").value=cells[4].innerText;
document.getElementById("fund").value=cells[5].innerText;

row.remove();
monitor();
}

// 📊 MONITOR
function monitor(){

let rows=document.querySelectorAll("#ppaTable tr");
let total=rows.length-1;

let done=0,ongoing=0,pipeline=0,forImpl=0,delayed=0;

for(let i=1;i<rows.length;i++){
let status=rows[i].cells[4].innerText;
let time=rows[i].cells[3].innerText;

if(status=="Completed") done++;
if(status=="Ongoing") ongoing++;
if(status=="Pipeline") pipeline++;
if(status=="For Implementation") forImpl++;
if(time=="Delayed" || time=="Not Implemented") delayed++;
}

document.getElementById("total").innerText=total;
document.getElementById("completed").innerText=done;
document.getElementById("ongoing").innerText=ongoing;
document.getElementById("pipelineCount").innerText=pipeline;
document.getElementById("forImpl").innerText=forImpl;
document.getElementById("delayed").innerText=delayed;

drawChart(done,ongoing,pipeline,forImpl,delayed);
}

// 📊 CHART
let chart;

function drawChart(done,ongoing,pipeline,forImpl,delayed){

let ctx=document.getElementById("chart");

if(chart) chart.destroy();

chart=new Chart(ctx,{
type:"bar",
data:{
labels:["Done","Ongoing","Pipeline","For Impl","Delayed"],
datasets:[{
label:"PPA Status",
data:[done,ongoing,pipeline,forImpl,delayed]
}]
}
});
}

// ☁️ SAVE TO GOOGLE SHEET
function saveToSheet(code,desc,budget,time,status,fund){

fetch(API_URL,{
method:"POST",
body:JSON.stringify({
code,desc,budget,time,status,fund
})
});

}

</script>

</body>
</html>
