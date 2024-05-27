<script lang="ts">
  import * as Tone from "tone";
  import { Midi } from "@tonejs/midi";

  import Song from "./assets/song.mid";
  import Game from "./Game.svelte";
  import type { Note } from "@tonejs/midi/dist/Note";

  // const synth = new Tone.Synth().toDestination();
  const sampler = new Tone.Sampler({
    urls: {
      A1: "A1.mp3",
      A2: "A2.mp3",
    },
    baseUrl: "https://tonejs.github.io/audio/casio/",
  }).toDestination();
  const synth = new Tone.PolySynth(Tone.Synth, {
    envelope: {
      attack: 0.02,
      decay: 0.1,
      sustain: 0.3,
      release: 1,
    },
  }).toDestination();

  let midi: Midi;

  async function loadSong() {
    midi = await Midi.fromUrl(Song);
    console.log("Loaded midi", midi);

    for (let track of midi.tracks) {
      for (let note of track.notes) {
        if (note.time + note.duration > trackLength) {
          trackLength = note.time + note.duration;
        }
      }
    }
  }

  // const CHANGE = 256;
  let currentSeconds = 0;
  let trackLength: number = 0;

  async function playNote(ms: number) {
    try {
      let notesToPlay: Note[] = [];

      for (let track of midi.tracks) {
        for (let note of track.notes) {
          if (note.time + note.duration >= currentSeconds && note.time + note.duration <= currentSeconds + ms / 1000) {
            console.log("Playing note", note);
            notesToPlay.push(note);
          }
        }
      }

      // get first note
      let firstNote = notesToPlay[0];
      for (let note of notesToPlay) {
        if (note.time < firstNote.time) {
          firstNote = note;
        }
      }

      for (let note of notesToPlay) {
        sampler.triggerAttackRelease(note.name, firstNote.duration, Tone.now() + (note.time - firstNote.time));
      }
    }
    finally {
      currentSeconds += ms / 1000;
      currentSeconds %= trackLength;
      if (currentSeconds < ms / 1000) {
        currentSeconds = 0;
      }

      console.log("Current seconds", currentSeconds);
    }
  }

  let load = loadSong();
</script>

<main>
  {#await load}
    <p>Loading...</p>
  {:then}
    <Game {playNote} />
  {/await}
</main>

<style>
  main {
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
  }
</style>
