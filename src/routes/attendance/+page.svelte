<script lang="ts">
  import Semid from "./semid.svelte";
  import { invoke } from "@tauri-apps/api/core";
  import { Store } from "@tauri-apps/plugin-store";
  import { selsemid, loading } from "./store.svelte";
  import { getContext } from "svelte";
  import Smallatten from "./smallatten.svelte";
  import { goto } from "$app/navigation";
  import { page } from "$app/state";
  function setReloadTime() {
    setTimeout(() => {
      if (page.url.pathname == "/attendance") {
        (async () => await getattendance())();
      }
    }, 60000);
  }

  let attendance_before: string | undefined = $state(undefined);
  let distime: undefined | number = $state(0);
  interface relaod {
    [key: string]: boolean;
  }
  interface datasate {
    [key: string]: string | undefined;
  }
  let reload: relaod = getContext("reload");
  let errors: datasate = $state(getContext("errors"));

  async function loadfromstorage() {
    const store = await Store.load("attendance.json");
    if (selsemid.value != undefined) {
      attendance_before = await store.get(`full_attendance_${selsemid.value}`);
      //console.log("sem id from storage",selsemid.value)
      let last_update: undefined | number = await store.get(
        `full_attendance_${selsemid.value}_lastupdate`,
      );
      distime = last_update;
    }
  }
  const time_diff_relaod = 10;
  function unixTimestamp() {
    return Math.floor(Date.now() / 1000);
  }

  async function getattendance() {
    // console.log("in get attendance")
    if (selsemid.value == undefined) {
      return;
    }
    let sel_sem = selsemid.value;
    if (loading.value.includes(sel_sem)) {
      return;
    }
    loading.value.push(sel_sem);
    const store = await Store.load("attendance.json");
    let last_update: undefined | number = await store.get(
      `full_attendance_${sel_sem}_lastupdate`,
    );
    distime = last_update;
    setReloadTime();

    if (
      (attendance_before == undefined ||
        last_update == undefined ||
        Math.abs(unixTimestamp() - last_update) > time_diff_relaod ||
        reload.status) &&
      errors.code != "stop"
    ) {
      reload.status = true;

      //@ts-ignore
      const [status, full_attendance_fetched] = await invoke("attendance", {
        semid: sel_sem,
      });
      reload.status = false;
      if (status && full_attendance_fetched != "") {
        if (errors.msg == "NE") {
          errors.msg = undefined;
        }
        const time = unixTimestamp();
        await store.set(`full_attendance_${sel_sem}_lastupdate`, time);
        distime = time;
        if (
          full_attendance_fetched != "" &&
          full_attendance_fetched != undefined &&
          full_attendance_fetched != attendance_before
        ) {
          await store.set(
            `full_attendance_${sel_sem}`,
            full_attendance_fetched,
          );
          await store.save();
          if (sel_sem == selsemid.value) {
            attendance_before = full_attendance_fetched;
          }
        }
      } else {
        if (full_attendance_fetched == "NE") {
          if (errors.code != "stop") {
            errors.code = "stop";
            errors.msg = "NE";
          }
          //triggerInfo("No Internet")
        } else {
          if (errors.code != "stop") {
            errors.code = "stop";
          }
          //triggerInfo(full_attendance1)
        }
      }
    }
    try {
      let index = loading.value.indexOf(sel_sem);
      if (index > -1) {
        loading.value.splice(index, 1);
      }
    } catch {}
  }
  $effect(() => {
    errors.code;
    //$inspect("att", loading.value);
    if (selsemid.value != undefined) {
      (async () => {
        await loadfromstorage();
        await getattendance();
      })();
    }
  });
</script>

<div class="">
  <div class="">
    <div class="">
      <Semid />
      <div>
        {#if attendance_before != undefined}
          <Smallatten attendance={attendance_before} updatedTime={distime} />
        {:else if errors.msg == "NE"}
          <p>Oops! No Internet Connection Detected</p>
        {:else if errors.msg == "NC"}
          <p>
            Please enter your credentials by navigating to the <button
              class=" link-primary"
              onclick={() => {
                goto("/settings/Credentials");
              }}>settings</button
            > menu.
          </p>
        {:else}
          <div class="skeleton h-[80vh] w-full"></div>
        {/if}
      </div>
    </div>
  </div>
</div>
