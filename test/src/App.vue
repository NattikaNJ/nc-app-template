<template>
  <!--
    SPDX-FileCopyrightText: Sebastian Stöcker <sebastian.stoecker@uni-marburg.de>
    SPDX-License-Identifier: AGPL-3.0-or-later
    -->
  <div id="content" class="app-test">
    <div id="container">
      <div id="content-left">
        <h3 style="font-weight: bold; margin-left: 10px">Attributliste</h3>

        <div id="attribute-list">
          <ul>
            <li v-for="(item, index) in resText" :key="index">
              <input
                type="checkbox"
                :id="index"
                :value="
                  item[
                    'Main.Daten.Metadaten.Metadata Repository.Code.Metadata RepositoryClass_attribut_name'
                  ]
                "
                v-model="selected"
              />
              <label>{{
                item[
                  "Main.Daten.Metadaten.Metadata Repository.Code.Metadata RepositoryClass_attribut_name"
                ]
              }}</label>
              <br />
            </li>
            <br />
          </ul>
        </div>
        <div id="selected-attribute">
          <p v-for="(item, index) in selected" :key="index">{{ item }}</p>
        </div>
      </div>
      <div id="content-right"></div>
      <p>Content Right</p>
      <!-- <button id="loadBtn" v-on:click="loadcsv">Load CSV file</button> -->
    </div>
  </div>
</template>

<script>
import ActionButton from "@nextcloud/vue/dist/Components/ActionButton";
import AppContent from "@nextcloud/vue/dist/Components/AppContent";
import AppNavigation from "@nextcloud/vue/dist/Components/AppNavigation";
import AppNavigationItem from "@nextcloud/vue/dist/Components/AppNavigationItem";
import AppNavigationNew from "@nextcloud/vue/dist/Components/AppNavigationNew";

import "@nextcloud/dialogs/styles/toast.scss";
import { generateUrl } from "@nextcloud/router";
import { showError, showSuccess } from "@nextcloud/dialogs";
import axios from "@nextcloud/axios";
import Papa from "papaparse";
import { ref } from "vue";

export default {
  name: "App",
  components: {
    ActionButton,
    AppContent,
    AppNavigation,
    AppNavigationItem,
    AppNavigationNew,
  },
  data() {
    return {
      notes: [],
      currentNoteId: null,
      updating: false,
      loading: true,
      resText: "",
      selected: ref([]),
    };
  },

  computed: {
    /**
     * Return the currently selected note object
     * @returns {Object|null}
     */
    currentNote() {
      if (this.currentNoteId === null) {
        return null;
      }
      return this.notes.find((note) => note.id === this.currentNoteId);
    },

    /**
     * Returns true if a note is selected and its title is not empty
     * @returns {Boolean}
     */
    savePossible() {
      return this.currentNote && this.currentNote.title !== "";
    },
  },
  /**
   * Fetch list of notes when the component is loaded
   */
  async mounted() {
    try {
      const response = await axios.get(generateUrl("/apps/test/notes"));
      this.notes = response.data;
    } catch (e) {
      console.error(e);
      showError(t("notestutorial", "Could not fetch notes"));
    }
    this.loading = false;
  },

  methods: {
    /**
     * Create a new note and focus the note content field automatically
     * @param {Object} note Note object
     */
    loadcsv() {
      const response = fetch(
        "http://localhost:8080/apps-extra/test/csvfile/diz.csv"
      )
        .then((res) => res.text())
        .then((text) => Papa.parse(text, { header: true }))
        .catch((e) => console.error(e));
      response.then((v) => (this.resText = Object.values(v.data)));
      console.log(response);
      return response;
    },
    openNote(note) {
      if (this.updating) {
        return;
      }
      this.currentNoteId = note.id;
      this.$nextTick(() => {
        this.$refs.content.focus();
      });
    },
    /**
     * Action tiggered when clicking the save button
     * create a new note or save
     */
    saveNote() {
      if (this.currentNoteId === -1) {
        this.createNote(this.currentNote);
      } else {
        this.updateNote(this.currentNote);
      }
    },
    /**
     * Create a new note and focus the note content field automatically
     * The note is not yet saved, therefore an id of -1 is used until it
     * has been persisted in the backend
     */
    newNote() {
      if (this.currentNoteId !== -1) {
        this.currentNoteId = -1;
        this.notes.push({
          id: -1,
          title: "",
          content: "",
        });
        this.$nextTick(() => {
          this.$refs.title.focus();
        });
      }
    },
    /**
     * Abort creating a new note
     */
    cancelNewNote() {
      this.notes.splice(
        this.notes.findIndex((note) => note.id === -1),
        1
      );
      this.currentNoteId = null;
    },
    /**
     * Create a new note by sending the information to the server
     * @param {Object} note Note object
     */
    async createNote(note) {
      this.updating = true;
      try {
        const response = await axios.post(
          generateUrl("/apps/test/notes"),
          note
        );
        const index = this.notes.findIndex(
          (match) => match.id === this.currentNoteId
        );
        this.$set(this.notes, index, response.data);
        this.currentNoteId = response.data.id;
      } catch (e) {
        console.error(e);
        showError(t("notestutorial", "Could not create the note"));
      }
      this.updating = false;
    },
    /**
     * Update an existing note on the server
     * @param {Object} note Note object
     */
    async updateNote(note) {
      this.updating = true;
      try {
        await axios.put(generateUrl(`/apps/test/notes/${note.id}`), note);
      } catch (e) {
        console.error(e);
        showError(t("notestutorial", "Could not update the note"));
      }
      this.updating = false;
    },
    /**
     * Delete a note, remove it from the frontend and show a hint
     * @param {Object} note Note object
     */
    async deleteNote(note) {
      try {
        await axios.delete(generateUrl(`/apps/test/notes/${note.id}`));
        this.notes.splice(this.notes.indexOf(note), 1);
        if (this.currentNoteId === note.id) {
          this.currentNoteId = null;
        }
        showSuccess(t("test", "Note deleted"));
      } catch (e) {
        console.error(e);
        showError(t("test", "Could not delete the note"));
      }
    },
  },
  // call function on page load
  // or beforeMount() {}
  created() {
    this.loadcsv();
  },
};
</script>
<style scoped>
#container {
  display: flex;
  flex-wrap: wrap;
  width: 100%;
  background-color: white;
}

#content-left {
  width: 20%;
  height: 90%;
  margin: 20px;
}

#content-left #attribute-list,
#content-left #selected-attribute {
  display: flex;
  flex-direction: column;
  overflow: scroll;
  scroll-margin: 10px;
  height: 50%;
  background-color: aliceblue;
  border-radius: 26px;
  padding: 20px;
  margin-bottom: 20px;
}

#content-right {
  margin: 10px;
}

#app-content > div {
  width: 100%;
  height: 100%;
  padding: 20px;
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}

input[type="text"] {
  width: 100%;
}

textarea {
  flex-grow: 1;
  width: 100%;
}

li {
  display: flex;
  column-gap: 8px;
  height: 55px;
}

li input {
  position: relative;
  top: -8px;
}
</style>
