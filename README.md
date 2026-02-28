index.html
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Geo-Tag Photo Portal (Free Version)</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/exif-js/2.3.0/exif.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf-autotable/3.5.25/jspdf.plugin.autotable.min.js"></script>
  <style>
  body {
    margin: 0;
    font-family: Arial, sans-serif;
    background-image: url("https://images.unsplash.com/photo-1500530855697-b586d89ba3ee");
    background-size: cover;
    background-position: center;
    background-attachment: fixed;
  }

  .container {
    background: rgba(255, 255, 255, 0.9);
    max-width: 800px;
    margin: 80px auto;
    padding: 30px;
    border-radius: 15px;
    text-align: center;
  }

  h1 {
    margin-bottom: 20px;
  }
</style>
</head>
<body>
  <div class="container">
    <h2>Geo-Tag Photo Work Portal (Free + Approx Road Distance)</h2>

    <input type="file" id="photoInput" accept="image/*" multiple>
    <div id="photoList"></div>

    <div id="distanceInfo"></div>

    <button id="openRouteBtn" style="display:none;">Open Connected Route</button>
    <button id="generatePDFBtn" style="display:none;">Generate PDF Report</button>

    <div id="mapPreview"></div>
  </div>

  <script>
    // === FULL JS CODE ===
    document.addEventListener("DOMContentLoaded", function () {

      let photosData = []; // {file, coordinates, datetime, thumbnail}

      const photoInput = document.getElementById("photoInput");
      const photoList = document.getElementById("photoList");
      const openRouteBtn = document.getElementById("openRouteBtn");
      const generatePDFBtn = document.getElementById("generatePDFBtn");
      const mapPreview = document.getElementById("mapPreview");
      const distanceInfo = document.getElementById("distanceInfo");

      function convertDMSToDD(deg, min, sec, dir) {
        let dd = deg + min/60 + sec/3600;
        if (dir === "S" || dir === "W") dd *= -1;
        return dd;
      }

      function haversineDistance(coord1, coord2) {
        const R = 6371;
        const [lat1, lon1] = coord1.split(',').map(Number);
        const [lat2, lon2] = coord2.split(',').map(Number);
        const dLat = (lat2 - lat1) * Math.PI/180;
        const dLon = (lon2 - lon1) * Math.PI/180;
        const a = Math.sin(dLat/2)**2 +
                  Math.cos(lat1*Math.PI/180)*Math.cos(lat2*Math.PI/180)*
                  Math.sin(dLon/2)**2;
        const c = 2*Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
        return R * c;
      }

      function computeApproxRoadDistance() {
        if(photosData.length<2) return 0;
        let total = 0;
        for(let i=0;i<photosData.length-1;i++){
          total += haversineDistance(photosData[i].coordinates, photosData[i+1].coordinates) * 1.3;
        }
        return total.toFixed(2);
      }

      function updateDistanceInfo(){
        const totalDistance = computeApproxRoadDistance();
        distanceInfo.innerHTML=`Total Approx Road Distance: ${totalDistance} KM`;
      }

      function updateMapPreview(){
        if(photosData.length<1){ mapPreview.innerHTML=""; return;}
        const coords = photosData.map(p=>p.coordinates);
        const url = `https://www.google.com/maps/dir/${coords.join('/')}/@${coords[0]},15z/data=!3m1!4b1!4m2!4m1!3e0`;
        mapPreview.innerHTML=`<iframe src="${url}" width="100%" height="100%" style="border:0;" allowfullscreen></iframe>`;
      }

      function renderPhotoList(){
        photoList.innerHTML="";
        photosData.forEach((p,index)=>{
          const div=document.createElement("div");
          div.innerHTML=`Photo ${index+1}: ${p.coordinates} | ${p.datetime || "-"} <button data-index="${index}">Delete</button>`;
          photoList.appendChild(div);
        });

        photoList.querySelectorAll("button").forEach(btn=>{
          btn.addEventListener("click", function(){
            const i=parseInt(this.dataset.index);
            photosData.splice(i,1);
            renderPhotoList();
            updateMapPreview();
            updateDistanceInfo();
            openRouteBtn.style.display=photosData.length>=2?"inline-block":"none";
            generatePDFBtn.style.display=photosData.length>=2?"inline-block":"none";
          });
        });
      }

      function createThumbnail(file, callback){
        const reader = new FileReader();
        reader.onload = function(e){
          const img = new Image();
          img.onload=function(){
            const canvas=document.createElement("canvas");
            const maxSize=60;
            let w=img.width;
            let h=img.height;
            if(w>h){
              h=h*(maxSize/w);
              w=maxSize;
            }else{
              w=w*(maxSize/h);
              h=maxSize;
            }
            canvas.width=w;
            canvas.height=h;
            const ctx=canvas.getContext("2d");
            ctx.drawImage(img,0,0,w,h);
            callback(canvas.toDataURL("image/jpeg"));
          };
          img.src=e.target.result;
        };
        reader.readAsDataURL(file);
      }

      photoInput.addEventListener("change", function(e){
        const files = Array.from(e.target.files);
        files.forEach(file=>{
          EXIF.getData(file,function(){
            const lat = EXIF.getTag(this,"GPSLatitude");
            const latRef = EXIF.getTag(this,"GPSLatitudeRef");
            const lng = EXIF.getTag(this,"GPSLongitude");
            const lngRef = EXIF.getTag(this,"GPSLongitudeRef");
            const datetime = EXIF.getTag(this,"DateTimeOriginal")||"-";

            if(lat && lng){
              const latitude = convertDMSToDD(lat[0].numerator/lat[0].denominator,
                                              lat[1].numerator/lat[1].denominator,
                                              lat[2].numerator/lat[2].denominator,
                                              latRef);
              const longitude = convertDMSToDD(lng[0].numerator/lng[0].denominator,
                                               lng[1].numerator/lng[1].denominator,
                                               lng[2].numerator/lng[2].denominator,
                                               lngRef);
              createThumbnail(file,function(th){
                photosData.push({file, coordinates:`${latitude},${longitude}`, datetime, thumbnail: th});
                renderPhotoList();
                updateMapPreview();
                updateDistanceInfo();
                openRouteBtn.style.display=photosData.length>=2?"inline-block":"none";
                generatePDFBtn.style.display=photosData.length>=2?"inline-block":"none";
              });
            }
          });
        });
        photoInput.value="";
      });

      openRouteBtn.addEventListener("click",function(){
        const coords = photosData.map(p=>p.coordinates);
        const url=`https://www.google.com/maps/dir/?api=1&origin=${coords[0]}&destination=${coords[coords.length-1]}&waypoints=${coords.slice(1,-1).join('|')}`;
        window.open(url,"_blank");
      });

      generatePDFBtn.addEventListener("click",function(){
        const { jsPDF }=window.jspdf;
        const doc=new jsPDF();
        doc.text("Geo-Tag Photo Work Report",14,20);

        const data = photosData.map((p,index)=> [p.thumbnail, index+1, p.datetime, p.coordinates]);

        doc.autoTable({
          head:[["Photo","#","Timestamp","Coordinates"]],
          body:data.map(d=>{
            return [{image:d[0], width:20, height:20, cellPadding: 5}, d[1], d[2], d[3]];
          }),
          startY:30,
          styles:{
            cellPadding:4,
            fontSize:10,
            valign:'middle',
            overflow:'linebreak'
          },
          columnStyles:{
            0:{cellWidth:25} // thumbnail column width
          },
          didDrawCell: function(data){
            if(data.column.index===0 && data.cell.section==="body"){
              doc.addImage(data.cell.raw.image,'JPEG',data.cell.x+3,data.cell.y+2,18,18);
            }
          },
          margin: {top:10, bottom:10}
        });

        doc.text(`Total Approx Road Distance: ${computeApproxRoadDistance()} KM`,14,doc.lastAutoTable.finalY+10);
        doc.save("GeoTag_Work_Report.pdf");
      });

    });
  </script>
</body>
</html>
](https://leandrotecson45-glitch.github.io/GeoTagPortal/)
