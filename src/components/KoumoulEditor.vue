<template>
    <div class="product-sheets-editor">
        <div class="options-header">
            <v-row>
                <p>Rechercher</p>
                <input type="text" v-model="search" />
            </v-row>
            <v-row>
                <p>Recherche inversée</p>
                <input type="text" v-model="searchReverse" />
            </v-row>
            <v-row v-if="currentView.name === 'textfields'">
                <p>Remplacer</p>
                <input id="replacetext-find" type="text" />
                <p class="ml-2">par</p>
                <input id="replacetext-replace" type="text" />
                <button class="button ml-5 mb-5" @click="findAndReplaceText()">Remplacer</button>
            </v-row>
            <v-row>
                <button class="button" @click="copySheetsInClipBoard()">Copier les fiches dans le presse-papier</button>
                <button class="button ml-5" @click="pasteSheetsFromText()">Coller la liste des fiches en JSON</button>
            </v-row>
            <v-row class="mt-5"> <button class="button" @click="switchView()">Changer de vue</button></v-row>
        </div>
        <div class="loader-container" v-if="loading">
            <div class="lds-roller">
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
                <div></div>
            </div>
        </div>
        <div v-if="currentView.showTextfields">
            <div class="product-sheet" v-for="(sheet, key) in sheets" :key="key">
                <div v-if="isInSearch(sheet.name)">
                    <h4>{{ sheet.name }}</h4>
                    <textarea :id="sheet.name" :value="JSON.stringify(sheet.content, null, 8)"></textarea>
                </div>
            </div>
        </div>
        <div v-if="currentView.showEditor">
            <div class="product-sheets-list" v-if="!sheet">
                <div style="display: inline-block" v-for="(sheet, key) in sheets" :key="key">
                    <div class="sheetFile" @click="selectSheet(sheet)" v-if="isInSearch(sheet.name)">
                        <v-icon class="icon" color="blue" x-large>fa fa-file</v-icon>
                        <p>{{ sheet.name }}</p>
                    </div>
                </div>
            </div>
            <div class="product-sheet" v-else>
                <div class="sheet-name">
                    <h1>{{ sheet.name }}</h1>
                    <div>
                        <v-icon @click="flexDirection = 'row'" class="icon mr-5" color="white" medium>fa-grip-lines-vertical</v-icon>
                        <v-icon @click="flexDirection = 'column'" class="icon mr-5" color="white" medium>fa-grip-lines</v-icon>
                        <v-icon @click="sheet = null" class="icon" color="white" large>fa-times</v-icon>
                    </div>
                </div>
                <div class="sheets-tabs-container" :style="'flexDirection: ' + flexDirection">
                    <div class="product-sheet-tab" v-for="(tab, tabId) in sheet.content.schema.allOf" :key="tabId">
                        <div class="tab-title-container">
                            <h3 class="sheet-tab-name">Onglet: {{ getRef(sheet.content.schema, tab.title) }}</h3>
                            <div class="icons">
                                <v-icon class="icon" color="white" small @click="editTabName(tabId)">fa fa-edit</v-icon>
                                <v-icon class="icon" color="white" small @click="addProperty(tabId)">fa fa-plus</v-icon>
                                <v-icon class="icon" color="white" small @click="removeTab(tabId)">fa fa-times-circle</v-icon>
                            </div>
                        </div>
                        <div style="display: inline-block" v-for="(propertyKey, propIndex) in Object.keys(tab.properties)" :key="propIndex">
                            <div
                                    class="product-sheet-property"
                                    :style="tab.properties[propertyKey]['title'] ? '' : 'display: none;'"
                                    @mousemove="itemHover($event)"
                                    :id="'tab' + tabId + '-index' + propIndex"
                            >
                                <div class="product-sheet-property-title-container">
                                    <p class="product-sheet-property-title">
                                        {{ getRef(sheet.content.schema, tab.properties[propertyKey].title) }}
                                    </p>
                                    <div>
                                        <v-icon class="icon" color="black" small @click="renameProperty(sheet.name, tabId, propertyKey)"
                                        >fa fa-edit</v-icon
                                        >
                                        <v-icon class="icon" color="black" small @click="deleteProperty(sheet.name, tabId, propertyKey)"
                                        >fa fa-times</v-icon
                                        >
                                        <v-icon @mousedown="grabItem($event, tabId, propertyKey)" class="icon grab" color="black" small
                                        >fa-arrows-alt</v-icon
                                        >
                                    </div>
                                </div>

                                <select @change="changePropertyType(sheet.name, tabId, propertyKey, $event)" name="type" id="type-select">
                                    <option
                                            v-for="(type, key) in types"
                                            :key="key"
                                            :value="type.value"
                                            :selected="tab.properties[propertyKey].type === type.value"
                                    >{{ type.name }}</option
                                    >
                                </select>
                                <div class="d-flex flex-column" v-if="propertyFetchesDriveList(tab.properties[propertyKey])">
                                    <a :href="tab.properties[propertyKey]['x-fromUrl']" target="_blank"
                                    >Liste des choix dans le drive
                                        {{ tab.properties[propertyKey].type == "array" ? "(choix multiple)" : "(choix unique)" }}</a
                                    >
                                    <input
                                            placeholder="Copiez l'url de la liste ici"
                                            class="url-input"
                                            :value="tab.properties[propertyKey]['x-fromUrl']"
                                    />
                                </div>
                                <div v-else-if="tab.properties[propertyKey].type == 'array'">
                                    <div v-if="tab.properties[propertyKey]['items']">
                                        <h5>Liste à choix multiple</h5>
                                        <div
                                                class="sheet-list-item"
                                                v-for="(listItem, itemId) in tab.properties[propertyKey]['items']['enum']"
                                                :key="itemId"
                                        >
                                            <input
                                                    class="sheet-list-item-input"
                                                    @keyup="updateListItem(sheet.name, tabId, propertyKey, itemId, 'items/enum', $event)"
                                                    :value="getRef(sheet.content.schema, listItem)"
                                                    placeholder="Ajouter un élément..."
                                            />
                                            <p
                                                    class="sheet-list-item-remove"
                                                    @click="removeListItem(sheet.name, tabId, propertyKey, itemId, 'items/enum')"
                                            >
                                                X
                                            </p>
                                        </div>
                                        <p
                                                class="sheet-list-item-add ml-2"
                                                @click="addListItem(sheet.name, tabId, propertyKey, 'items/enum')"
                                        >
                                            Ajouter un choix
                                        </p>
                                    </div>
                                </div>
                                <div v-if="tab.properties[propertyKey]['enum']">
                                    <h5>Liste à choix unique</h5>
                                    <div
                                            class="sheet-list-item"
                                            v-for="(listItem, itemId) in tab.properties[propertyKey]['enum']"
                                            :key="itemId"
                                    >
                                        <input
                                                class="sheet-list-item-input"
                                                :value="getRef(sheet.content.schema, listItem)"
                                                @keyup="updateListItem(sheet.name, tabId, propertyKey, itemId, 'enum', $event)"
                                                placeholder="Ajouter un élément..."
                                        />
                                        <p
                                                class="sheet-list-item-remove"
                                                @click="removeListItem(sheet.name, tabId, propertyKey, itemId, 'enum')"
                                        >
                                            X
                                        </p>
                                    </div>
                                    <p class="sheet-list-item-add ml-2" @click="addListItem(sheet.name, tabId, propertyKey, 'enum')">
                                        Ajouter un choix
                                    </p>
                                </div>
                                <v-checkbox
                                        class="sheet-checkbox"
                                        :input-value="propertyFetchesDriveList(tab.properties[propertyKey])"
                                        @change="togglePropertyFetchesDriveList(sheet.name, tabId, propertyKey)"
                                        label="Choisir parmis une liste du drive"
                                ></v-checkbox>
                                <v-checkbox
                                        class="sheet-checkbox"
                                        :input-value="propertyIsUniqueChoiceList(tab.properties[propertyKey])"
                                        @change="toggleSheetUniqueChoiceList(sheet.name, tabId, propertyKey)"
                                        label="Liste à choix unique"
                                ></v-checkbox>
                                <v-checkbox
                                        class="sheet-checkbox"
                                        v-model="tab.properties[propertyKey].readOnly"
                                        label="Grisé"
                                ></v-checkbox>
                                <v-checkbox
                                        v-if="specificities.adminOnly"
                                        class="sheet-checkbox"
                                        :input-value="propertyIsAdminOnly(tab.properties[propertyKey])"
                                        label="Modifiable uniquement par les admins"
                                        @change="toggleSheetPropertyAdminOnly(sheet.name, tabId, propertyKey)"
                                ></v-checkbox>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <p v-if="sheet && currentView.name === 'editor'" class="add-tab-button" @click="addTab()">
            Ajouter un onglet
        </p>
<!--        <p class="save-button" @click="saveSheets()">Sauvegarder</p>-->
    </div>
</template>
<script>
import "@koumoul/vjsf/lib/VJsf.css";
import schema from "@koumoul/vjsf/lib/utils/schema";
export default {
    name: "editProductsSheets",
    computed: {
        schema() {
            return schema;
        },
    },
    props: {},
    data: () => ({
        specificities: {
            adminOnly: false,
        },
        itemPreview: null,
        mouseX: 0,
        mouseY: 0,
        grabbedItemNewPosition: null,
        grabbedItem: null,
        grabbedItemClone: null,
        flexDirection: "row",
        loading: true,
        searchReverse: "",
        sheets: [],
        sheet: null,
        originalSheets: [],
        search: "",
        views: [
            {
                name: "textfields",
                showTextfields: true,
                getEditedFiles: (originals) => {
                    const editedFiles = [];

                    document.querySelectorAll(".product-sheets-editor textarea").forEach(textarea => {
                        const originalFile = originals.find(file => file.name === textarea.id);

                        const originalContent = originalFile.content;
                        let newContent;

                        try {
                            newContent = JSON.parse(textarea.value);
                        } catch (e) {
                            alert("JSON invalide pour la fiche " + textarea.id + ", voir console");
                            return console.log("Erreur fiche " + textarea.id, e);
                        }

                        if (JSON.stringify(originalContent) !== JSON.stringify(newContent)) {
                            editedFiles.push({
                                name: textarea.id,
                                content: JSON.parse(textarea.value),
                            });
                        }
                    });

                    return editedFiles;
                },
            },
            {
                name: "editor",
                showEditor: true,
                getEditedFiles: (originals, currents) =>
                    currents.filter(current => {
                        const original = originals.find(e => e.name === current.name);
                        return JSON.stringify(original) !== JSON.stringify(current);
                    }),
            },
        ],
        currentView: {},
        types: [
            {
                value: "string",
                name: "Chaîne de caractères",
            },
            {
                value: "number",
                name: "Nombre",
            },
            {
                value: "boolean",
                name: "Case à cocher",
            },
            {
                value: "array",
                name: "Liste",
            },
        ],
    }),
    watch: {},
    methods: {
        addToObjectAtIndex(obj, key, value, index) {

            // Create a temp object and index variable
            const temp = {};
            let i = 0;

            // Loop through the original object
            for (let prop in obj) {
                if (obj.hasOwnProperty(prop)) {
                    // If the indexes match, add the new item
                    if (i === index && key && value) {
                        temp[key] = value;
                    }

                    // Add the current item in the loop to the temp obj
                    temp[prop] = obj[prop];

                    // Increase the count
                    i++;
                }
            }

            // If no index, add to the end
            if (!index && key && value) {
                temp[key] = value;
            }

            return temp;
        },
        itemHover(event) {
            if (this.grabbedItem) {
                const scrollY = window.scrollY;
                const scrollX = window.scrollX;

                const mouseY = this.mouseY - scrollY;
                const mouseX = this.mouseX - scrollX;

                const item = event.target.closest(".product-sheet-property");
                const viewportOffset = item.getBoundingClientRect();
                const top = viewportOffset.top;
                const left = viewportOffset.left;
                const height = item.offsetHeight;
                const width = item.offsetWidth;

                const isInLeft = mouseX > left && mouseX < left + width / 2;
                const isInRight = !isInLeft;
                const isInTop = mouseY > top && mouseY < top + height / 2;
                const isInBottom = !isInTop;
                //                console.log(isInLeft ? "left" : "", isInRight ? "right" : "", isInTop ? "top" : "", isInBottom ? "bottom" : "");

                let appendBefore = this.flexDirection === "row" ? isInTop : isInLeft;
                let appendAfter = this.flexDirection === "row" ? isInBottom : isInRight;

                //example id: tab1-index0
                const positionInfos = item.id.split("-");
                const tabIndex = parseInt(positionInfos[0].split("tab")[1]);
                let propertyIndex = parseInt(positionInfos[1].split("index")[1]);

                if (appendBefore) {
                    this.itemPreview.remove();
                    item.before(this.itemPreview);
                }

                if (appendAfter) {
                    this.itemPreview.remove();
                    item.after(this.itemPreview);
                    //propertyIndex += 1;
                }

                this.grabbedItemNewPosition = { tabIndex, propertyIndex };
            }
        },
        grabItem(event, tabId, propertyKey) {
            this.grabbedItem = {
                tabId: tabId,
                propertyKey: propertyKey,
                div: event.target.closest(".product-sheet-property"),
            };
            console.log("grabbedItem", this.grabbedItem);
        },
        editTabName(tabId) {
            const tab = this.sheet.content.schema.allOf[tabId];
            const newTabName = prompt("Nom de l'onglet", this.getRef(this.sheet.content.schema, tab.title));
            if (newTabName) {
                tab.title = newTabName;
                this.updateSheet();
            }
        },
        addTab() {
            const newTabName = prompt("Nom de l'onglet");
            if (newTabName) {
                this.sheet.content.schema.allOf.push({
                    id: this.sheet.content.schema.allOf.length + 1,
                    properties: {},
                    title: newTabName,
                    type: "object",
                });
            }
        },
        removeTab(tabId) {
            const tab = this.sheet.content.schema.allOf[tabId];
            const confirmation = window.confirm("Supprimer l'onglet " + this.getRef(this.sheet.content.schema, tab.title) + " ?");
            if (confirmation) {
                this.sheet.content.schema.allOf.splice(tabId, 1);
                this.updateSheet();
            }
        },
        addProperty(tabId) {
            const tab = this.sheet.content.schema.allOf[tabId];
            const newPropertyKey = prompt("Clé de la nouvelle propriété (ex: PersonName)");

            if (newPropertyKey) {
                const newPropertyName = prompt("Nom de la nouvelle propriété (ex: Nom de la personne)");
                if (newPropertyName) {
                    tab.properties[newPropertyKey] = {
                        title: newPropertyName,
                        type: "string",
                    };
                    this.updateSheet();
                }
            }
        },
        deleteProperty(sheetName, tabId, propertyKey) {
            const sheet = this.sheets.filter(e => e.name === sheetName)[0];
            const property = this.getProperty(sheetName, tabId, propertyKey);
            const confirmation = window.confirm("Supprimer " + this.getRef(sheet.content.schema, property.title) + " ?");

            if (confirmation) {
                delete sheet.content.schema.allOf[tabId].properties[propertyKey];
            }

            this.updateSheet();
        },
        renameProperty(sheetName, tabId, propertyKey) {
            const sheet = this.sheets.filter(e => e.name === sheetName)[0];
            const property = this.getProperty(sheetName, tabId, propertyKey);
            const newName = prompt("Changer le nom de la propriété", this.getRef(sheet.content.schema, property.title));
            const newKey = prompt("Changer la clé ?", propertyKey);

            if (newName) {
                property.title = newName;
            }

            if (newKey && newKey != propertyKey) {
                delete sheet.content.schema.allOf[tabId].properties[propertyKey];
                sheet.content.schema.allOf[tabId].properties[newKey] = property;
            }

            this.updateSheet();
        },
        updateSheet() {
            const sheet = this.sheet;
            this.sheet = null;
            this.sheet = sheet;
        },
        selectSheet(sheet) {
            const sheetReference = this.sheets.find(e => e.name === sheet.name);
            this.sheet = sheetReference;
        },
        disableMultipleChoicesItems(property) {
            if (property["items"]) {
                property["items_disabled"] = property["items"];
                delete property["items"];
            }
        },
        enableMultipleChoicesItems(property) {
            //retrieve old value
            if (property["items_disabled"]) {
                property["items"] = property["items_disabled"];
                delete property["items_disabled"];
            }

            //create list with empty item if needed
            const items = property["items"];
            if (!items || !items.enum || !items.enum.length) {
                property["items"] = {
                    enum: ["",],
                    type: "string",
                };
            }
        },
        disableUniqueChoicesItems(property) {
            if (property["enum"]) {
                property["enum_disabled"] = property["enum"];
                delete property["enum"];
            }
        },
        enableUniqueChoicesItems(property) {
            this.disableDriveList(property);
            if (property.type === "array") {
                property.type = "string";
            }
            if (property["enum_disabled"]) {
                property["enum"] = property["enum_disabled"];
                delete property["enum_disabled"];
            } else {
                property["enum"] = ["",];
            }
        },
        changePropertyType(sheetName, tabId, propertyKey, event) {
            const property = this.getProperty(sheetName, tabId, propertyKey);
            property.type = event.target.value;

            if (property.type == "array") {
                this.disableUniqueChoicesItems(property);
                this.enableMultipleChoicesItems(property);
            } else {
                this.disableMultipleChoicesItems(property);
            }

            this.updateSheet();
        },
        toggleSheetUniqueChoiceList(schemaName, tabId, propertyName) {
            const sheet = this.sheets.filter(e => e.name === schemaName)[0];
            let property = sheet.content.schema.allOf[tabId].properties[propertyName];

            if (this.propertyIsUniqueChoiceList(property)) {
                this.disableUniqueChoicesItems(property);

                if (property.type === "array") {
                    this.enableMultipleChoicesItems(property);
                }
            } else {
                this.disableMultipleChoicesItems(property);
                this.enableUniqueChoicesItems(property);
            }

            this.updateSheet();
        },
        propertyFetchesDriveList(property) {
            return property.hasOwnProperty("x-fromUrl");
        },
        enableDriveList(property) {
            this.disableUniqueChoicesItems(property);
            if (property["x-fromUrl_disabled"]) {
                property["x-fromUrl"] = property["x-fromUrl_disabled"];
                delete property["x-fromUrl_disabled"];
            } else {
                property["x-fromUrl"] = "";
            }
        },
        disableDriveList(property) {
            property["x-fromUrl_disabled"] = property["x-fromUrl"];
            delete property["x-fromUrl"];
        },
        togglePropertyFetchesDriveList(schemaName, tabId, propertyName) {
            let property = this.sheet.content.schema.allOf[tabId].properties[propertyName];

            if (this.propertyFetchesDriveList(property)) {
                this.disableDriveList(property);
            } else {
                this.enableDriveList(property);
            }

            this.updateSheet();
        },
        toggleSheetPropertyAdminOnly(schemaName, tabId, propertyName) {
            const sheet = this.sheets.filter(e => e.name === schemaName)[0];
            let property = sheet.content.schema.allOf[tabId].properties[propertyName];
            if (!property["x-class"]) {
                property["x-class"] = "";
            }

            if (property["x-class"].includes("adminOnly")) {
                property["x-class"] = property["x-class"].replaceAll("adminOnly", "");
            } else {
                property["x-class"] += " adminOnly";
            }
        },
        propertyIsUniqueChoiceList(property) {
            return property["enum"] && property["enum"].length;
        },
        propertyIsAdminOnly(property) {
            const className = property["x-class"];

            if (className) {
                if (className.includes("adminOnly")) {
                    return true;
                }
            }

            return false;
        },
        getProperty(schemaName, tabId, propertyName, extraPath = "") {
            const sheet = this.sheets.filter(e => e.name === schemaName)[0];
            let property = sheet.content.schema.allOf[tabId].properties[propertyName];

            if (extraPath) {
                const keys = extraPath.split("/");
                for (const k of keys) {
                    property = property[k];
                }
            }

            return property;
        },
        addListItem(schemaName, tabId, propertyName, extraPath) {
            const list = this.getProperty(schemaName, tabId, propertyName, extraPath);
            list.push("");
            this.updateSheet();
        },
        updateListItem(schemaName, tabId, propertyName, itemId, extraPath, event) {
            const list = this.getProperty(schemaName, tabId, propertyName, extraPath);
            list[itemId] = event.target.value;
            this.updateSheet();
        },
        removeListItem(schemaName, tabId, propertyName, itemId, extraPath) {
            let list = this.getProperty(schemaName, tabId, propertyName, extraPath);
            list = list.splice(itemId, 1);
            this.updateSheet();
        },
        getRef(schema, title) {
            if (typeof title == "string") {
                return title;
            }
            if (!title) {
                return "non défini";
            }
            const key = title["$ref"].split("~$locale~/")[1];
            const keys = key.split("/");

            let value = schema["i18n"]["fr"];
            for (const k of keys) {
                value = value[k];
            }

            return value;
        },
        switchView() {
            this.sheets = JSON.parse(JSON.stringify(this.originalSheets));

            let currentIndex = this.views.indexOf(this.currentView);
            if (currentIndex < this.views.length - 1) {
                currentIndex++;
            } else {
                currentIndex = 0;
            }

            this.currentView = this.views[currentIndex];
        },
        pasteSheetsFromText() {
            const stringifiedSheets = prompt("Collez les fiches ici", "");

            try {
                const newSheets = JSON.parse(stringifiedSheets);
                if (Array.isArray(newSheets)){
                    this.sheets = newSheets;
                }else{
                    this.sheets = [newSheets];
                }
            } catch (error) {
                alert("JSON INVALIDE");
            }
        },
        isInSearch(sheetname) {
            if (this.searchReverse.length && sheetname.toLowerCase().includes(this.searchReverse.toLowerCase())) {
                return false;
            }

            if (this.search.length && sheetname.toLowerCase().includes(this.search.toLowerCase())) {
                return true;
            } else {
                return !this.search.length;
            }
        },
        copySheetsInClipBoard() {
            const textToCopy = JSON.stringify(this.sheets, null, 3)

            if (navigator && navigator.clipboard && navigator.clipboard.writeText){
                navigator.clipboard.writeText(textToCopy);
            }else{
                const el = document.createElement('textarea');
                el.value = textToCopy;
                el.setAttribute('readonly', '');
                el.style.position = 'absolute';
                el.style.left = '-9999px';
                document.body.appendChild(el);
                el.select();
                document.execCommand('copy');
                document.body.removeChild(el);
            }

        },
        saveSheets() {
            const editedFiles = this.currentView.getEditedFiles(this.originalSheets, this.sheets);
            console.log("Edited files", editedFiles);
            this.saveSheetsIntoDrive(editedFiles);
        },
        findAndReplaceText() {
            const textToFind = document.getElementById("replacetext-find").value;
            const textToReplace = document.getElementById("replacetext-replace").value;

            var textareas = document.getElementsByTagName("textarea");

            for (var i = 0; i < textareas.length; i++) {
                var textarea = textareas[i];
                var currentText = textarea.value;

                // Perform the find and replace operation
                var newText = currentText.replace(new RegExp(textToFind, "g"), textToReplace);

                // Update the textarea value with the replaced text
                textarea.value = newText;
            }
        },
        async saveSheetsIntoDrive(files) {
            // console.log("sending ", files);
            // const formData = new FormData();
            // formData.append("sheets", JSON.stringify(files));
            //
            // const saveRequest = await this.$axios.post(
            //     "/pim/categories/productssheets/save?bucket=" + configEnv.bucket + "&path=Private/PIM/JSF",
            //     formData,
            // );
            // console.log("Response", saveRequest);
            // alert("Sauvegarde réussie");
        },
    },
    async mounted() {
        this.itemPreview = document.createElement("div");
        this.itemPreview.className = "itemPreview";
        const draggedItem = document.getElementById("grabbedItemClone");

        if (draggedItem) {
            draggedItem.remove();
        }

        this.currentView = this.views.find(view => view.name === "editor");

        //const sheetsRequest = await this.$axios.get("/pim/categories/productssheets?bucket=" + configEnv.bucket + "&path=Private/PIM/JSF");
        this.sheets = [{name: 'Exemple.json', content: {schema: { allOf: [{title: 'Onglet 1', properties: {PersonName:{title: 'Nom de la personne', type:'string'}}}] }}}];//sheetsRequest.data.filter(e => e.content["schema"]);
        this.originalSheets = JSON.parse(JSON.stringify(this.sheets));
        console.log("SHEETS", this.sheets);
        this.loading = false;

        document.onmousemove = e => {
            if (this.grabbedItem) {
                this.mouseX = e.pageX;
                this.mouseY = e.pageY;

                if (!this.grabbedItemClone) {
                    this.grabbedItemClone = this.grabbedItem.div.cloneNode(true);
                    this.grabbedItemClone.id = "grabbedItemClone";
                }
                document.body.appendChild(this.grabbedItemClone);
                this.grabbedItem.div.style.opacity = 0.5;
                this.grabbedItemClone.style.position = "absolute";
                this.grabbedItemClone.style.top = e.pageY + "px";
                this.grabbedItemClone.style.left = e.pageX + "px";
                this.grabbedItemClone.style.transform = "translate(-100%, 0%)";
            }
        };

        document.onmouseup = () => {
            if (this.grabbedItem) {
                console.log("GRABBED OBJECT", this.grabbedItem);
                console.log("grabbedItemNewPosition", this.grabbedItemNewPosition);

                const grabbedPropertyKey = this.grabbedItem.propertyKey;
                const grabbedPropertyTab = this.sheet.content.schema.allOf[this.grabbedItem.tabId];
                const grabbedProperty = JSON.parse(JSON.stringify(grabbedPropertyTab.properties[grabbedPropertyKey]));
                delete this.sheet.content.schema.allOf[this.grabbedItem.tabId].properties[grabbedPropertyKey];
                console.log("grabbedProperty", grabbedProperty);

                const goalTab = this.sheet.content.schema.allOf[this.grabbedItemNewPosition.tabIndex];
                //goalTab;
                //.splice(position, 0, valueToAdd);

                let newProperties;

                console.log(this.grabbedItemNewPosition.propertyIndex, Object.keys(goalTab.properties).length);
                if (this.grabbedItemNewPosition.propertyIndex < Object.keys(goalTab.properties).length) {
                    newProperties = this.addToObjectAtIndex(
                        goalTab.properties,
                        grabbedPropertyKey,
                        grabbedProperty,
                        this.grabbedItemNewPosition.propertyIndex,
                    );
                } else {
                    newProperties = goalTab.properties;
                    newProperties[grabbedPropertyKey] = grabbedProperty;
                }

                console.log("taking " + this.grabbedItem.propertyKey + " from tab " + this.grabbedItem.tabId);
                console.log("to tab " + this.grabbedItemNewPosition.tabIndex + " position " + this.grabbedItemNewPosition.propertyIndex);

                this.grabbedItem.div.style.opacity = 1;
                this.itemPreview.remove();

                console.log("newProperties", newProperties);
                this.sheet.content.schema.allOf[this.grabbedItemNewPosition.tabIndex].properties = newProperties;

                this.updateSheet();
            }
            this.grabbedItem = null;
            this.grabbedItemClone = null;

            const draggedItem = document.getElementById("grabbedItemClone");

            if (draggedItem) {
                draggedItem.remove();
            }
        };
    },
};
</script>

<style>

.v-main__wrap{
    background-color: whitesmoke;
}

.icon.grab {
    cursor: grab;
}

.icon.grab:active {
    cursor: grabbing;
}

.product-sheets-editor{
    margin-top: 1%;
    padding: 20px;
}

.product-sheet{
    margin-top: 20px;
}

.product-sheets-editor input {
    background-color: white;
    border: solid 1px black;
    margin-bottom: 20px;
    margin-left: 10px;
}

.product-sheets-editor textarea {
    min-width: 90vw !important;
    width: 90vw !important;
    background-color: white !important;
}

.product-sheets-editor textarea::-webkit-scrollbar {
    width: 16px;
    height: 100px;
}

.product-sheets-editor textarea::-webkit-scrollbar-thumb {
    height: 100px !important;
}

.button {
    background-color: #1e88dd;
    color: white;
    border-radius: 5px;
    padding: 5px 10px;
    cursor: pointer;
}

.product-sheets-editor .save-button,
.product-sheets-editor .add-tab-button {
    background-color: #464be8;
    color: white;
    border-radius: 5px;
    padding: 5px 10px;
    position: fixed;
    bottom: 0;
    right: 0;
    font-size: 17px;
    font-weight: bold;
    text-shadow: 0px 0px 3px black;
    cursor: pointer;
}

.product-sheets-editor .add-tab-button {
    right: 10px;
}

.options-header {
    margin-left: 10px;
}

.product-sheets-editor select {
    border: solid 1px;
    margin-bottom: 5px;
    padding: 5px 10px;
    border-radius: 3px;
}

.product-sheet-property {
    background-color: white;
    border: solid 1px darkcyan;
    border-radius: 5px;
    margin: 5px;
    padding: 10px;
    width: fit-content;
    display: inline-block;
}

.product-sheet-tab {
    border: solid 2px darkslateblue;
    border-radius: 3px;
    padding: 10px;
    width: 100%;
}

.product-sheet-property-title {
    font-weight: bold;
    letter-spacing: 2px;
}

.sheet-name {
    margin-top: 10px;
    background-color: darkslateblue;
    color: white;
    padding: 5px 10px;
    border-radius: 5px;
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0;
    transform: translateY(2px);
    display: flex;
    justify-content: space-between;
}

.quit-sheet {
    color: white;
    font-size: 20px;
}

.tab-title-container {
    background-color: #1e88dd;
    display: flex;
    width: fit-content;
    border-radius: 5px;
    padding: 5px 10px;
}

.tab-title-container .icons {
    margin-left: 20px;
}

.tab-title-container .icon {
    margin-left: 10px;
}

.sheet-tab-name {
    color: white;
    width: fit-content;
    border-radius: 5px;
}

.sheets-tabs-container {
    display: flex;
}

.sheet-list-item {
    display: flex;
}
.sheet-list-item-remove {
    background-color: #ef5350;
    color: white;
    font-size: 13px;
    font-weight: bold;
    width: 23px;
    height: 23px;
    line-height: 23px;
    aspect-ratio: 1/1;
    text-align: center;
    border-radius: 50%;
    margin-left: 10px;
    cursor: pointer;
}

.sheet-list-item-input {
    border: solid 1px lightblue !important;
    border-radius: 5px;
    padding: 5px 10px;
}

.sheet-list-item-add {
    color: black;
    width: fit-content;
    padding: 5px 10px;
    border-radius: 5px;
    border: solid 1px #1e88dd;
    cursor: pointer;
}

.sheet-checkbox {
    margin-top: 5px;
    margin-bottom: -22px;
}

a{
    text-decoration: none;
}

.v-label{
    color: black !important;
}

.url-input {
    border-radius: 3px;
    width: 95%;
    transform: translateX(-3.5%);
    margin-top: 5px;
    padding: 5px 5px;
}

.product-sheets-list{
    margin-top: 20px;
}

.product-sheets-list .sheetFile {
    background-color: white;
    width: fit-content;
    min-width: 140px;
    max-width: 140px;
    aspect-ratio: 1/1;
    border-radius: 5px;
    margin: 5px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-direction: column;
    font-weight: bold;
    text-align: center;
    cursor: pointer;
    transition: transform 0.2s;
}

.product-sheets-list .sheetFile p {
    font-size: 13px;
    line-height: 14px;
    word-break: break-all;
    letter-spacing: 2px;
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 80%;
}

.product-sheets-list .sheetFile .icon {
    transform: translateY(65%) scale(1.05);
}

.product-sheets-list .sheetFile:hover {
    transform: scale(1.05);
}

.product-sheet-property-title-container {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: space-between;
}

.product-sheet-property-title-container .icon {
    transform: translateY(-10px);
    margin-left: 10px;
}

.itemPreview {
    border: dashed 2px black;
    opacity: 0.7;
    border-radius: 5px;
    width: 310px;
    height: 200px;
    margin-left: 5px;
}

/* LOADER */

.loader-container {
    width: 100vw;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 30vh;
}

.lds-roller {
    display: inline-block;
    position: relative;
    width: 80px;
    height: 80px;
}
.lds-roller div {
    animation: lds-roller 1.2s cubic-bezier(0.5, 0, 0.5, 1) infinite;
    transform-origin: 40px 40px;
}
.lds-roller div:after {
    content: " ";
    display: block;
    position: absolute;
    width: 7px;
    height: 7px;
    border-radius: 50%;
    background: black;
    margin: -4px 0 0 -4px;
}
.lds-roller div:nth-child(1) {
    animation-delay: -0.036s;
}
.lds-roller div:nth-child(1):after {
    top: 63px;
    left: 63px;
}
.lds-roller div:nth-child(2) {
    animation-delay: -0.072s;
}
.lds-roller div:nth-child(2):after {
    top: 68px;
    left: 56px;
}
.lds-roller div:nth-child(3) {
    animation-delay: -0.108s;
}
.lds-roller div:nth-child(3):after {
    top: 71px;
    left: 48px;
}
.lds-roller div:nth-child(4) {
    animation-delay: -0.144s;
}
.lds-roller div:nth-child(4):after {
    top: 72px;
    left: 40px;
}
.lds-roller div:nth-child(5) {
    animation-delay: -0.18s;
}
.lds-roller div:nth-child(5):after {
    top: 71px;
    left: 32px;
}
.lds-roller div:nth-child(6) {
    animation-delay: -0.216s;
}
.lds-roller div:nth-child(6):after {
    top: 68px;
    left: 24px;
}
.lds-roller div:nth-child(7) {
    animation-delay: -0.252s;
}
.lds-roller div:nth-child(7):after {
    top: 63px;
    left: 17px;
}
.lds-roller div:nth-child(8) {
    animation-delay: -0.288s;
}
.lds-roller div:nth-child(8):after {
    top: 56px;
    left: 12px;
}
@keyframes lds-roller {
    0% {
        transform: rotate(0deg);
    }
    100% {
        transform: rotate(360deg);
    }
}
</style>
