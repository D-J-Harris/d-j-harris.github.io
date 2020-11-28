---
layout: page
title: WordArt
permalink: /wordart/
---

<body onload="init()">
    Select Image
    <br><input type="file" id="uploadImageInput" onchange="getBaseUrl()" accept="image/x-png,image/jpeg">
    <br>Type Text
    <br><textarea id="myTextArea" rows="3" cols="80" placeholder="Your text here"></textarea>
    <input type="number" id="myNumberArea" rows="1" cols="4" placeholder="Repetitions of text">
    <br><button type="button" id="sendPostButton" onclick="sendPost()">Send Post</button>
    <div id="placehere"></div>
    <button onclick="window.location.reload();">Refresh Page</button>
</body>

<script type="text/javascript">
var baseString = undefined
function init() {
}


function getBaseUrl ()  {
    var file = document.querySelector('input[type=file]')['files'][0];
    var reader = new FileReader();
    reader.onloadend = function () {
        baseString = reader.result;
    };
    reader.readAsDataURL(file);
}


function sendPost(){
    document.getElementById("sendPostButton").disabled = true;
    sendToServer(baseString, document.getElementById("myTextArea").value.repeat(document.getElementById("myNumberArea").value));
}


function sendToServer(image, text){

    const xhr = new XMLHttpRequest();
    xhr.onreadystatechange = function () {
        if (this.readyState != 4) return;
        if (this.status == 200) {
            var data = JSON.parse(this.responseText);

            var image_tag = data['image']
            var myimg = document.createElement("img");
            myimg.src = image_tag;
            console.log(myimg)
            document.getElementById("placehere").appendChild(myimg);
        }
        else{
            alert("Unexpected error... try again with perhaps different text or number of repetitions");
        }
        document.getElementById("sendPostButton").disabled = false;
    };
    const url = "http://dharris.pythonanywhere.com//calculate";
    xhr.open("POST", url, true);
    xhr.setRequestHeader('Content-Type', 'application/json');
    const sendData = JSON.stringify({
        image: image,
        text: text
    })
    xhr.send(sendData);
}
</script>
