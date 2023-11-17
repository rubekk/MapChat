<script>
    import { onMount } from "svelte";
    import { initializeApp } from "firebase/app";
	import { getDatabase, ref, set, push, onValue, onDisconnect, child } from "firebase/database";
    import { firebaseConfig } from "$lib/firebaseConfig";
    import "./../app.css";
    import messageSound from "$lib/message.mp3";

    const firebaseApp=initializeApp(firebaseConfig);
    const db=getDatabase();

    let leaflet,
        map,
        tile,
        chatMarkerGroup,
        activeMarkerGroup,
        myLocation,
        msgDiv, 
        defaultMarker,
        activeMarker,
        audio,
        msg="",
        errorMsg="",
        userId="",
        activeUsers=0,
        tileLayer="default";

    let showMyLocation=false,
        showWhatInfo=false,
        showActiveUsers=false,
        mapLoaded=false,
        increasedUsers=false,
        showMsg=false,
        showError=false;
        
    let userCoords=[],
        activeUsersCoords=[],
        defaultViewCoords=['27.7172', '85.3240'],
        markers=[],
        chatData=[],
        days=[];
    
    // get user location
    const getLocation=()=>{
        if (navigator.geolocation) {
            navigator.geolocation.getCurrentPosition(
                position=> {
                    userCoords= [ position.coords.latitude.toFixed(4), position.coords.longitude.toFixed(4) ]
                },
                ()=> {
                    errorMsg="Sorry! your location couldn't be tracked. Your location will be set to a random location. Please use different device / browser for exact location.";
                    showError=true;

                    userCoords=['27.7172', '85.3240']

                    setTimeout(()=> showError=false, 7500);
                }
            );
        } 
        else {
            errorMsg="Sorry! your location couldn't be tracked. Your location will be set to a random location. Please use different device / browser for exact location.";
            showError=true;

            userCoords=['27.7172', '85.3240']

            setTimeout(()=> showError=false, 7500);
        }
    }

    // send message
    const sendMessage=()=>{
        if(msg.length>1 && userCoords.length>0){
            const chatListRef = ref(db, 'chat');
            const newChatRef = push(chatListRef);
            
            let currentDate=new Date();
            let data={
                msg: msg,
                time: `${String(currentDate.getHours()).length==1?'0'+String(currentDate.getHours()):currentDate.getHours()}:${String(currentDate.getMinutes()).length==1?'0'+String(currentDate.getMinutes()):currentDate.getMinutes()}:${String(currentDate.getSeconds()).length==1?'0'+String(currentDate.getSeconds()):currentDate.getSeconds()}`,
                date: `${currentDate.getFullYear()}/${currentDate.getMonth()+1}/${currentDate.getDate()}`,
                coords: userCoords
            };
    
            set(newChatRef, data);

            msg="";

            audio.play();
        }
        else if(userCoords.length==0) getLocation();
    }

    // increase active users
    const increaseActiveUsers=()=>{  
        if(userCoords.length==0) getLocation();

        if(!increasedUsers){
            const activeRef = ref(db, 'active');
            const newActiveRef = push(activeRef);
            
            
            set(newActiveRef, {
                uid: Math.floor(Math.random()*500)
            })
            
            userId=newActiveRef.key;
            
            onDisconnect(ref(db,'active/' + userId)).set({
                uid: null
            })
        }
        
        increasedUsers=true;
    }

    // handle individual message click
    const handleMessageClick=index=>{
        let includes=false;
        let includeIndex=0;

        markers.forEach((elem,i)=>{
            // if(elem.coords[0]==chatData[index].coords[0] && elem.coords[1]==chatData[index].coords[1]) {
            if(elem.coords[0].substring(0, elem.coords[0].length-1)==chatData[index].coords[0].substring(0, chatData[index].coords[0].length-1) && elem.coords[1].substring(0, elem.coords[1].length-1)==chatData[index].coords[1].substring(0, chatData[index].coords[1].length-1)) {
                includes=true;
                includeIndex=i;
            }
        })

        if(!includes) markers.push(chatData[index])
        else markers[includeIndex]=chatData[index];

        showMarkers(chatData[index].coords, includeIndex);
    }

    const handleInputClick=()=>{
        if(userCoords.length==0) getLocation();
    }

    // show markers on message load
    const showMarkersOnMessageLoad=()=>{
        days=[];
        
        chatData.forEach(elem=>{
            let includes=false;
            let includeIndex=0;

            markers.forEach((item,i)=>{
                // if(elem.coords[0]==item.coords[0] && elem.coords[1]==item.coords[1]) {
                if(elem.coords[0].substring(0, elem.coords[0].length-1)==item.coords[0].substring(0, item.coords[0].length-1) && elem.coords[1].substring(0, elem.coords[1].length-1)==item.coords[1].substring(0, item.coords[1].length-1)) {
                    includes=true;
                    includeIndex=i;
                }
            })

            if(!includes) markers.push(elem);
            else markers[includeIndex]=elem;
        })

        showMarkers();
    }

    // show markers
    const showMarkers=(viewCoords=defaultViewCoords, index=-1)=>{
        if(markers.length==0) return; 

        map.removeLayer(chatMarkerGroup);
        chatMarkerGroup=leaflet.layerGroup().addTo(map);

        if(index>=markers.length) index=markers.length-1;
        else if(index==-1 && markers.length>0 && chatData.length>0){
            markers.forEach((elem,i)=>{
                if(elem.coords[0]==chatData[chatData.length-1].coords[0] && elem.coords[1]==chatData[chatData.length-1].coords[1])  index=i;
            })
        }

        markers.forEach((elem,i)=>{
            if(index==i){
                leaflet.marker(elem.coords, {icon: defaultMarker}).addTo(chatMarkerGroup).bindPopup(leaflet.popup().setContent(
                    `<div>
                        <p style="font-style: italic; font-size: .8rem"> ${elem.msg} </p>
                        <p style="margin: -17px 0; float: right; font-style: italic; font-size: .6rem; color: #797979;"> ${elem.time} </p>
                    </div>`
                    )).openPopup();
                }
            else{
                leaflet.marker(elem.coords, {icon: defaultMarker}).addTo(chatMarkerGroup).bindPopup(leaflet.popup().setContent(
                    `<div>
                        <p style="font-style: italic; font-size: .8rem"> ${elem.msg} </p>
                        <p style="margin: -17px 0; float: right; font-style: italic; font-size: .6rem; color: #797979;"> ${elem.time} </p>
                    </div>`
                ));
            }
        })

        map.setView(viewCoords, 12);
    }

    // show active users
    const handleShowActiveUsers=()=>{
        showActiveUsers=!showActiveUsers;

        if(showActiveUsers && activeUsersCoords.length>0){
            map.removeLayer(activeMarkerGroup);
            activeMarkerGroup=leaflet.layerGroup().addTo(map);

            activeUsersCoords.forEach(elem=>{
                if(elem.indexOf("*")>0) leaflet.marker(elem.split("*"), {icon: activeMarker}).addTo(activeMarkerGroup)
            })
        }
        else{
            map.removeLayer(activeMarkerGroup);
        }
    }

    // show me
    const showMe=()=>{
        if(userCoords.length==0) getLocation();
        else {
            showMyLocation=!showMyLocation;
            map.removeLayer(myLocation);

            if(!showMyLocation) return;
            
            myLocation=leaflet.layerGroup().addTo(map);
            leaflet.marker(userCoords, {icon: activeMarker}).addTo(myLocation).bindPopup(leaflet.popup().setContent(
                `<div>
                    <p style="font-style: italic; font-size: .8rem"> You are here </p>
                </div>`
            )).openPopup();
            map.setView(userCoords, 13);
        }
    }

    // handle message dates
    const handleMessageDate=date=>{
        let day=date.substring(5,date.length);

        if(days.includes(day)) return false;
        else{
            days.push(day);
            return true;
        }
    }

    // handle show message
    const handleShowMsg=()=>{
        days=[];

        showMsg=!showMsg;

        setTimeout(()=>{
            if(showMsg && msgDiv) msgDiv.scrollTop = msgDiv.scrollHeight;
        }, 100)
    }

    // handle show what info
    const handleWhatInfo=()=>{
        showWhatInfo=!showWhatInfo;
    }

    // get day of week
    const getDayOfWeek=date=>{
        const dayOfWeek = new Date(date).getDay();    
        return isNaN(dayOfWeek) ? null : 
            ['Sunday', 'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday'][dayOfWeek];
    }

    // check if chat user is active or not
    const checkActiveChat=coords=>{
        if(activeUsersCoords.length==0) return false;

        let isActive=false;

        activeUsersCoords.forEach(elem=>{
            if(elem.indexOf("*")>0){

                let elemArr=elem.split("*");
                
                if(elemArr[0].substring(0, elemArr[0].length-1)==coords[0].substring(0, coords[0].length-1) && elemArr[1].substring(0, elemArr[1].length-1)==coords[1].substring(0, coords[1].length-1)) isActive=true;
            }
        })

        return isActive;
    }

    // change tile layer
    const handleTileLayer=()=>{
        map.removeLayer(tile);
        
        if(tileLayer=="default"){
            tileLayer="satellite";

            tile=leaflet.tileLayer('http://{s}.google.com/vt/lyrs=s&x={x}&y={y}&z={z}', {
                maxZoom: 20,
                subdomains:['mt0','mt1','mt2','mt3']
            }).addTo(map);
        }
        else{
            tileLayer="default";

            tile=leaflet.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);
        }
    }

    // keep upto date with all chat messages from firebase
    onValue(ref(db, "chat"), (snapshot) => {
        if(snapshot.val()){
            chatData = Object.values(snapshot.val());

            defaultViewCoords=chatData[chatData.length-1].coords;
        }
    });

    // keep upto date with active users from firebase
    onValue(ref(db, "active"), (snapshot) => {
        if(snapshot.val()){
            activeUsers=Object.values(snapshot.val());

            activeUsersCoords=[];
            activeUsers.forEach(elem=>{
                activeUsersCoords.push(elem.uid);
            })
        }
    });

    // draw map first
    onMount(async ()=>{
        leaflet= await import("leaflet");

        map = leaflet.map('map', { zoomControl: false }).setView(defaultViewCoords, 12);
        tile=leaflet.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png').addTo(map);

        // map.removeLayer(tile);

        leaflet.control.zoom({
            position: 'bottomright'
        }).addTo(map);

        chatMarkerGroup=leaflet.layerGroup().addTo(map);
        activeMarkerGroup=leaflet.layerGroup().addTo(map);
        myLocation=leaflet.layerGroup().addTo(map);
        
        defaultMarker = leaflet.divIcon({
            className: 'custom-div-icon',
            html: "<div class='default-marker-outer'><div class='default-marker'></div></div>",
            iconSize: [20, 20],
            iconAnchor: [10, 20],
            popupAnchor: [5, -7.5]
        });
        activeMarker = leaflet.divIcon({
            className: 'custom-div-icon',
            html: "<div class='active-marker-outer anim'><div class='active-marker'></div></div>",
            iconSize: [20, 20],
            iconAnchor: [10, 20],
            popupAnchor: [5, -7.5]
        });

        mapLoaded=true;
    })

    $: if(chatData && mapLoaded) showMarkersOnMessageLoad();

    $: if(userCoords.length>0) set(ref(db, 'active/' + userId), {
            uid: userCoords[0]+"*"+userCoords[1]
        });
</script>

<div class="container" on:click={increaseActiveUsers}>
    <div id="map"></div>

    <div class="title">
        <span class="title-map">Map</span><span class="title-chat">Chat</span>
        <br>
        <span class="sub-title">अब नक्शा मा कुरा</span>
    </div>

    <div class="active">Active users: {activeUsers? activeUsers.length: 0}</div>

    <div on:click={handleShowActiveUsers} class={showActiveUsers? 'show-active anim': 'show-active'}>Show active users</div>

    <div on:click={showMe} class={showMyLocation? 'show-me anim': 'show-me'} title="locate me">
        <i class="fa-solid fa-location-dot"></i>
    </div>

    <div on:click={handleWhatInfo} class={showWhatInfo? 'what anim': 'what'} title="what">
        <i class="fa-solid fa-question"></i>
    </div>

    <div on:click={handleTileLayer} class="tile-layer">
        <i class="fa-solid fa-repeat"></i>
    </div>
    

    <div class="chat">
        <i on:click={handleShowMsg} class={`fa-solid ${showMsg?'fa-chevron-down':'fa-chevron-up'} cross-chat`} title={showMsg?'minimize':'expand'}></i>
        {#if chatData.length>0 && showMsg}
        <div bind:this={msgDiv} class="message">
            {#each chatData as data,index}
                {#if handleMessageDate(data.date)}
                    <p class="msg-date">{getDayOfWeek(data.date)} {data.date}</p>
                {/if}
                <div on:click={()=>handleMessageClick(index)} class={userCoords.length>0 && data.coords[0].substring(0, data.coords[0].length-1)==userCoords[0].substring(0, userCoords[0].length-1) && data.coords[1].substring(0, data.coords[1].length-1)==userCoords[1].substring(0, userCoords[1].length-1)? 'ind-message right-side': 'ind-message'}>
                    <i class="fa-solid fa-chevron-right" title={checkActiveChat(data.coords)?"online":"offline"}></i>
                    <div class="msg">{data.msg}</div>
                    <div class="time">{data.time}</div>
                </div>
            {/each}
        </div>
        {/if}
        <div class="inp">
            <input bind:value={msg} on:click={handleInputClick} on:keydown={e=>{if(e.keyCode==13) sendMessage()}} type="text" placeholder="Enter a message">
            <button>
                <i on:click={sendMessage} class="fa-solid fa-paper-plane"></i>
            </button>
        </div>
    </div>

    {#if showWhatInfo}
    <div class="what-info">
        MapChat is a realtime location based chat website. Created by Rubek
    </div>
    {/if}

    {#if showError}
    <div class="error">
        {errorMsg}
    </div>
    {/if}

    <audio bind:this={audio} src={messageSound}></audio>
</div>
