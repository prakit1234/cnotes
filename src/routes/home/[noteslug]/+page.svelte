<script lang="ts">
  import { onMount } from "svelte";
  import {
    toasts,
    ToastContainer as ToastContainerAny,
    FlatToast as FlatToastAny,
  } from "svelte-toasts"; //imports toasts, toastContainer and flatToast to show toasts

  type DateFormat = "date" | "time" | "datetime";

  let data: any[] = [];
  let slug: string = "";
  let error: string = "";
  let createdDate: string;
  let createdTime: string;
  let displayTime: string;
  let selectedNote: {
    note_id: string;
    title: string;
    note_content: string;
    board: string;
    grade: string;
    school: string;
    subject: string;
  } | null = null;
  let isChanged: boolean = false;
  const ToastContainer = ToastContainerAny as any;
  const FlatToast = FlatToastAny as any;

  const showToast = (
    title: string,
    body: string,
    duration: number,
    type: string
  ) => {
    const toast = toasts.add({
      title: title,
      description: body,
      duration: duration,
      placement: "bottom-right",
      //@ts-ignore
      type: "info",
      theme: "dark",
      //@ts-ignore
      placement: "bottom-right",
      showProgress: true,
      //@ts-ignore
      type: type,
      //@ts-ignore
      theme: "dark",
      onClick: () => {},
      onRemove: () => {},
    });
  };

  function updateNote() {
    if (data.length > 0) {
      selectedNote = {
        note_id: data[0].note_id,
        title: data[0].title,
        note_content: data[0].note_content,
        board: data[0].board,
        grade: data[0].grade,
        school: data[0].school,
        subject: data[0].subject,
      };
      syncWithBackend(); // Call the sync function whenever the note is updated
    }
  }
  async function syncWithBackend() {
    if (selectedNote) {
      console.log("Syncing with backend...");
      try {
        const response = await fetch("/api/database", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            action: "updateNote",
            id: selectedNote.note_id,
            title: selectedNote.title,
            content: selectedNote.note_content,
            board: selectedNote.board,
            grade: selectedNote.grade,
            school: selectedNote.school,
            subject: selectedNote.subject,
          }),
        });
        const result = await response.json();
        if (result.status !== 200) {
          showToast("Error", "Failed to update note", 2500, "error");
        } else {
          console.log(result);
        }
      } catch (error) {
        console.error("Error updating note:", error);
      }
    }
  }
  async function getNoteFromDb(slug: string, email: string) {
    const response = await fetch("../../api/database/", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        action: "getNote",
        slug: slug,
        UserEmail: email,
      }),
    });
    const result = await response.json();
    if (result.status === 200) {
      if (result.message.length > 0) {
        data = result.message;
      } else {
        error = "You don't have permission to access this note.";
      }
    } else {
      error = "Error getting note from datatbase";
    }
  }
  function formatDate(
    dateString: string,
    format: DateFormat = "datetime"
  ):
    | string
    | {
        date: string;
        time: string;
        displayTime: string; // For displaying 12-hour format
        inputTime: string; // For HTML time input (24-hour format)
      } {
    const date = new Date(dateString);

    const pad = (num: number): string => num.toString().padStart(2, "0");

    // Date components
    const year = date.getFullYear();
    const month = pad(date.getMonth() + 1);
    const day = pad(date.getDate());

    // Time components
    const hours24 = date.getHours();
    const minutes = pad(date.getMinutes());

    // 12-hour format for display
    const hours12 = hours24 % 12 || 12;
    const ampm = hours24 >= 12 ? "PM" : "AM";
    const displayTime = `${pad(hours12)}:${minutes} ${ampm}`;

    // 24-hour format for input
    const inputTime = `${pad(hours24)}:${minutes}`;

    switch (format) {
      case "date":
        return `${year}-${month}-${day}`;
      case "time":
        return inputTime;
      case "datetime":
      default:
        return {
          date: `${year}-${month}-${day}`,
          time: displayTime,
          displayTime: displayTime, // 12-hour format (06:30 PM)
          inputTime: inputTime, // 24-hour format (18:30)
        };
    }
  }
  onMount(async () => {
    const userEmail = sessionStorage.getItem("Email");
    const localNotes = localStorage.getItem("notes");

    // slug = window.location.href.slice(27); // Development server
    slug = window.location.href.slice(30); // Production server

    if (userEmail) {
      if (localNotes) {
        data = JSON.parse(localNotes);
      }
      await getNoteFromDb(slug, userEmail);
      if (data && data[0] && data[0].date_created) {
        const result = formatDate(data[0].date_created);
        if (typeof result === "object") {
          createdDate = result.date;
          createdTime = result.inputTime;
          displayTime = result.displayTime;
        }
      }
    } else {
      error = "You must be logged in to view your notes";
    }
  });
</script>

<svelte:component this={ToastContainer} let:data>
  <svelte:component this={FlatToast} {data} />
</svelte:component>

{#if error}
  {error}
{:else if data.length > 0}
  <div class="note">
    <input
      type="text"
      class="text-lg font-bold edit-title"
      bind:value={data[0].title}
      on:input={() => {
        isChanged = true;
      }}
    /><br /><br />
    <div class="meta-data">
      <label>
        Board:
        <input
          type="text"
          bind:value={data[0].board}
          on:input={() => {
            isChanged = true;
          }}
        /><br />
      </label>
      <label>
        Created Date:
        <input
          type="date"
          bind:value={createdDate}
          on:input={() => {
            isChanged = true;
          }}
        />
      </label>
      <label>
        Grade:
        <input
          type="text"
          bind:value={data[0].grade}
          on:input={() => {
            isChanged = true;
          }}
        />
      </label>
      <label>
        School:
        <input
          type="text"
          bind:value={data[0].school}
          on:input={() => {
            isChanged = true;
          }}
        />
      </label>
      <label>
        Subject
        <input
          type="text"
          bind:value={data[0].subject}
          on:input={() => {
            isChanged = true;
          }}
        />
      </label>
      {#if isChanged}
        <button class="btn btn-outline btn-accent" on:click={updateNote}
          >Save</button
        >
      {:else}
        <button class="btn btn-outline btn-accent" disabled>Save</button>
      {/if}
    </div>
    <br />
    <textarea
      class="edit-content"
      bind:value={data[0].note_content}
      on:input={() => {
        isChanged = true;
      }}
    ></textarea>
  </div>
{:else}
  <div class="Loading"><h1>Loading Your Note...</h1></div>
{/if}

<style>
  .meta-data {
    display: flex;
    flex-direction: row;
    gap: 10px;
  }
  .note {
    padding: 5px;
  }
  textarea {
    width: 100%;
    height: 100vh;
  }
</style>
