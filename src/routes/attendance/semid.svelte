<script lang="ts">
  import { invoke } from "@tauri-apps/api/core";
  import { Store } from "@tauri-apps/plugin-store";
  import { selsemid } from "./store.svelte";
  import { getContext } from "svelte";
  import { untrack } from "svelte";

  interface relaod {
    [key: string]: boolean;
  }
  interface datasate {
    [key: string]: string | undefined;
  }
  let reload: relaod = getContext("reload");
  let errors: datasate = getContext("errors");
  let isCollapsed = $state(true);

  const time_diff_relaod = 10;
  function unixTimestamp() {
    return Math.floor(Date.now() / 1000);
  }

  let semid: string | undefined = $state(undefined);
  let lastUpdate: number | undefined = $state(undefined);

  async function loadfromstorage() {
    const store = await Store.load("attendance.json");
    let selsem: string | undefined = await store.get("sel_attendance_semid");
    semid = await store.get("attendance_semid");
    selsemid.value = selsem;
    await store.save();
    //console.log("from semid loadfrom storage",selsemid.value)
  }

  async function getsemids() {
    const store = await Store.load("attendance.json");
    let semids: string | undefined = await store.get("attendance_semid");
    lastUpdate = await store.get("attendance_semid_lastupdate");
    let check = semids == undefined ? true : false;
    if (
      (selsemid.value == undefined ||
        lastUpdate == undefined ||
        unixTimestamp() - lastUpdate > time_diff_relaod ||
        reload.status) &&
      errors.code != "stop"
    ) {
      if (check) {
        reload.status = true;
      }
      //@ts-ignore
      let [status, semid_rs] = await invoke("attendance_semid");
      //("updating semids")
      if (check) {
        reload.status = false;
      }
      if (status) {
        let temp = JSON.parse(semid_rs);
        store.set("attendance_semid", temp);
        let time = unixTimestamp();
        await store.set("attendance_semid_lastupdate", time);
        await store.save();
        //console.log("updating attendance semids set");
        if (temp != semid && temp != "") {
          semid = temp;
        }
      }
    }
  }
  //loadfromstorage().then(()=>{getsemids()})

  //@ts-ignore
  async function onClickHandel(id) {
    let selectedValue = id.target.value;

    const store = await Store.load("attendance.json");
    if (selectedValue != "" && selectedValue != undefined) {
      await store.set("sel_attendance_semid", selectedValue);
      await store.save();
      selsemid.value = selectedValue;
      //console.log("from semid",selsemid.value)
    }
  }

  let semname = $derived.by(() => {
    if (selsemid.value) {
    }
    if (semid == undefined) {
      getsemids();
      return "Loading";
    }
    if (selsemid.value == undefined) {
      return "Pick a semester";
    } else {
      if (semid == undefined) {
        getsemids();
        return "Loading";
      }
      for (let i = 0; semid.length > i; i++) {
        let k = semid[i].split(":");
        if (k[1] == selsemid.value) {
          return k[0];
        }
      }
    }
  });

  $effect(() => {
    errors.code;
    if (reload.status) {
    }
    // console.log("running effect from attendance semid");
    untrack(() => {
      loadfromstorage().then(() => getsemids());
    });
  });
  function toogleCollapsed() {
    isCollapsed = !isCollapsed;
  }
  function capitalizeFirstLetter(val) {
    try {
      var splitStr = val.toLowerCase().split(" ");
      for (var i = 0; i < splitStr.length; i++) {
        splitStr[i] =
          splitStr[i].charAt(0).toUpperCase() + splitStr[i].substring(1);
      }
      return splitStr.join(" ");
    } catch {
      return val;
    }
  }
</script>

<div class=" mb-3 max-h-[24vh] overflow-auto">
  <div
    tabindex="0"
    role="button"
    class="collapse collapse-arrow bg-base-100 border-base-300 border rounded-lg shadow-sm {isCollapsed
      ? 'collapse-close'
      : 'collapse-open'}"
  >
    <input type="checkbox" onclick={toogleCollapsed} />
    <div class="collapse-title font-semibold flex items-center p-4">
      <span class="mr-2">📋</span>
      {capitalizeFirstLetter(semname)}
    </div>
    <div class="collapse-content text-sm pb-3">
      <div class="flex flex-col gap-3 overflow-auto py-2 px-1">
        {#each semid as value}
          <div class="hover:bg-base-200 rounded-md px-3 py-2 transition-all duration-200">
            <input
              type="radio"
              name="radio-1"
              class="radio radio-sm radio-primary mr-2"
              id={value.split(":")[1]}
              value={value.split(":")[1]}
              bind:group={selsemid.value}
              onclick={async (e) => {
                isCollapsed = !isCollapsed;
                await onClickHandel(e);
              }}
            />
            <label for={value.split(":")[1]} class="cursor-pointer">
              {capitalizeFirstLetter(value.split(":")[0])}
            </label>
          </div>
        {/each}
      </div>
    </div>
  </div>
</div>
