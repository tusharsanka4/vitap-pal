<script lang="ts">
  import { User, KeyRound } from "lucide-svelte";
  import { Store } from "@tauri-apps/plugin-store";
  import { invoke } from "@tauri-apps/api/core";
  import { goto } from "$app/navigation";
  import { creds } from "../../globalstate.svelte";
  import { getContext } from "svelte";
  import { Github } from "lucide-svelte";

  interface relaod {
    [key: string]: boolean;
  }
  interface datasate {
    [key: string]: string | undefined;
  }

  let reload: relaod = getContext("reload");
  let errors: datasate = getContext("errors");

  let crederror: string | undefined = $state(undefined);

  let timeout = () =>
    setTimeout(() => {
      crederror = undefined;
    }, 3000);
  let password = $state("");
  let username = $state("");
  let running = $state(false);

  async function update() {
    running = true;
    const store = await Store.load("credentials.json");
    if (username == "" && password == "") {
      return;
    }
    for (let i = 0; i < 3; i++) {
      reload.status = true;
      //@ts-ignore
      const [status, k] = await invoke("try_login", {
        username: username,
        password: password,
      });
      reload.status = false;

      if (k == "Invalid Captcha") {
        continue;
      } else if (k == "SS") {
        if ((() => creds.username)() != username) {
          (await Store.load("timetable.json")).clear();
          (await Store.load("attendance.json")).clear();
          (await Store.load("marks.json")).clear();
          (await Store.load("examschedule.json")).clear();
        }
        await store.set("username", username);
        await store.set("password", password);
        await store.save();
        errors.code = undefined;
        creds.username = username;
        goto("/");
        //add toast
        break;
      } else if (k == "Invalid  Username/Password") {
        crederror = k;
        running = false;
        timeout();
      } else {
        if (k == "NE") {
          crederror =
            "No internet connection. Please connect to the internet and try again";
          timeout();
          running = false;
        } else if (k == "VE") {
          crederror = "VTOP is currently down. Please try again later.";
          timeout();
          running = false;
        } else {
          crederror = k;
          timeout();
        }
        break;
      }
    }

    running = false;
  }

  async function handelSubmit() {
    if (crederror != undefined) {
      return;
    }
    if (username != "" && password != "") {
      await update();
    }
  }

  //onDestroy(()=> reload.status=false)
</script>

<div class="flex flex-col min-h-[85vh]">
  <div class=" grow">
    <div>
      {#if errors.msg == "PC"}
        <div role="alert" class="alert alert-warning">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            class="h-6 w-6 shrink-0 stroke-current"
            fill="none"
            viewBox="0 0 24 24"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              stroke-width="2"
              d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
            />
          </svg>
          <span>It looks like you changed your password, please update it</span>
        </div>
      {/if}
    </div>

    <div class="card w-full max-w-md bg-base-100 p-6">
      <h2 class="text-center text-2xl font-semibold mb-6">Vtop Credentials</h2>
      <form>
        <div class="form-control mb-4">
          <label for="username" class="label">
            <User />
            <span class="label-text">Username</span>
          </label>
          <input
            type="text"
            id="username"
            placeholder="Enter your username"
            class="input input-bordered w-full"
            bind:value={username}
          />
        </div>

        <div class="form-control mb-4">
          <label for="password" class="label">
            <KeyRound />
            <span class="label-text">Password</span>
          </label>
          <input
            type="password"
            id="password"
            placeholder="Enter your password"
            class="input input-bordered w-full"
            bind:value={password}
          />
        </div>

        <!-- Submit Button -->
        <div class="form-control mt-6">
          <button
            type="submit"
            class="btn {crederror == undefined
              ? 'btn-primary'
              : 'btn-error'} w-full"
            onclick={handelSubmit}
            disabled={running}
          >
            {crederror == undefined
              ? running == true
                ? "updating.."
                : "update"
              : crederror}
          </button>
        </div>
      </form>
    </div>

    <div class="card bg-base-100 w-full shadow-sm">
      <!-- <div class="card-body justify-center items-center">
        <h2 class="card-title">Vtop Username</h2>
        <p>{(() => creds.username)()}</p>
      </div> -->
    </div>
  </div>
  <div class="w-full">
    <footer class=" flex items-center justify-between gap-4 p-4">
      <small class="opacity-70"
        >Made By <a
          href="https://discordapp.com/users/1147202428404519013"
          class="link-primary"
          target="_blank">Synaptic</a
        ></small
      >
      <small class="opacity-70">
        <a
          href="https://github.com/synapticsurge/vitap-pal"
          class="link-primary"
          target="_blank"><div class="flex"><Github />source code</div></a
        >
      </small>
      <small class="opacity-70"
        ><a
          href="https://linktr.ee/synapticsurge"
          class="link-primary"
          target="_blank">Talk to Dev</a
        >
      </small>
    </footer>
  </div>
</div>
