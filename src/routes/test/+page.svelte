<script>
	import { initializeApp } from "firebase/app";
    import { getDatabase, ref, get, set, push, onValue, child, onDisconnect } from "firebase/database";
    import { firebaseConfig } from "$lib/firebaseConfig";

    let activeUsers=0,
        increasedUsers=false, 
        userId="";

    const firebaseApp=initializeApp(firebaseConfig);
    const db=getDatabase();
    const activeRef = ref(db, 'active');

    const increaseActiveUsers=()=>{
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

    onValue(ref(db, "active"), (snapshot) => {
        if(snapshot.val()){
            activeUsers=Object.keys(snapshot.val());
        }
    });
</script>

<div on:click={increaseActiveUsers} class="active">
    Active users:  {activeUsers?activeUsers.length:0}
</div>