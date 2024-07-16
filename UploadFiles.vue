<template>
  <div class="upload-files">
    <div class="box borderinputfile" ref="borderinputfile">
      <div
        @dragenter="dragFile($event)"
        :class="!addMoreFile ? 'borderdanger' : ''"
        class="inputfilebase"
      >
        <span class="default-message" :style="!addMoreFile ? 'color:#d03827;' : ''">
          <template v-if="addMoreFile">
            <div class="structure" ref="structure">
              <svg
                xmlns="http://www.w3.org/2000/svg"
                enable-background="new 0 0 24 24"
                class="svg"
                viewBox="0 0 24 24"
              >
                <g>
                  <rect fill="none" height="24" width="24" />
                </g>
                <g>
                  <path
                    d="M18,15v3H6v-3H4v3c0,1.1,0.9,2,2,2h12c1.1,0,2-0.9,2-2v-3H18z M7,9l1.41,1.41L11,7.83V16h2V7.83l2.59,2.58L17,9l-5-5L7,9z"
                  />
                </g>
              </svg>
              <p class="body01">Drag and drop</p>
              <span @click="enableClick" class="btnfileinput caption3" role="button"
                >or <span class="caption3 ml-1"> choose your files</span></span
              >
            </div>
          </template>

          <template v-else> Please wait while the files are uploading. </template>
        </span>
      </div>
      <input
        class="w-100 h-100 mltiplefiles"
        @click="disabledCLick"
        style="opacity: 0"
        type="file"
        multiple
        ref="mltiplefiles"
        @change="onChangeFiles($event)"
        :accept="setListOfAuthFiles"
        @dragover="dragStyleEnter()"
        @dragleave="dragStyleLeave()"
      />
    </div>

    <div class="progressbase order-2" @click="removeActionSequence($event)"></div>

    <div class="modal-buttons order-3">
      <button class="btn third-button" @click="hideModalParent('.formcreatefiles')">
        Cancel
      </button>
      <button
        class="btn primary-button"
        @click="sendData(sendFilesList), showActionBox()"
        :disabled="disabled"
      >
        Save
      </button>
    </div>
  </div>
</template>

<script>
import store from "./../../index.js";

export default {
  name: "UploadFiles",
  props: {
    hideModalParent: {
      type: Function,
      required: true,
    },
    structureTosendDataParent: {
      type: Function,
      required: true,
    },
    baseurl: {
      type: String,
      required: true,
    },
    displayid: {
      type: String,
      required: true,
    },
    folderid: {
      type: String,
      required: false,
    },
    showActionBox: {
      type: Function,
      required: false,
    },
  },
  data: () => {
    return {
      gigabyte: 1000 * 1000 * 1000 * 2,
      listoffiletypes: [
        "image/png",
        "image/jpeg",
        "image/jpg",
        "text/plain",
        ".pdf",
        ".doc",
        ".docx",
        ".txt",
        ".png",
        ".jpeg",
        ".jpg",
        ".tif",
      ],
      setListOfAuthFiles: "",
      updateFileList: [],
      timeDefaultProgressBar: 100,
      fileprogressbartime: 0,
      progressBarElementList: [],
      addMoreFile: true,
      lastIndexFileList: 0,
      progressBarBackgroundColor: "#333",
      reorderFileList: [],
      sendFilesList: [],
      abortSequence: false,
      disabled: false,
      urlfilecreate: "",
      dataListForm: [],
      fileSizeAccept: true,
    };
  },
  methods: {
    dragStyleEnter() {
      if (!this.$refs.borderinputfile.classList.contains("active")) {
        this.$refs.borderinputfile.classList.add("active");
      }
      return;
    },
    dragStyleLeave() {
      if (this.$refs.borderinputfile.classList.contains("active")) {
        this.$refs.borderinputfile.classList.remove("active");
      }
      return;
    },
    toBase64(file) {
      let file64 = new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => resolve(reader.result);
        reader.onerror = (error) => reject(error);
      });

      return file64;
    },
    onChangeFiles(event) {
      if (event.target.nodeName === "INPUT" && event.target)
        //Remove Event Click from Input File
        this.$refs.mltiplefiles.classList.add("remove_click");
      console.log(event.target.nodeName);

      //Check all files progress before send a new file
      if (this.addMoreFile) {
        //Change abort boolValue
        this.abortSequence = false;

        //List Files when occured change - Get All Files -- Add Array Unique
        this.updateFileList = this.arrayUnique(this.$refs.mltiplefiles.files);

        //Check if file is authorizated by list
        let checkFilesExtension = this.updateFileList.filter((value) => {
          let cleanNameExt = value.name.split(".");

          //console.log(cleanNameExt[(cleanNameExt.length - 1)]);

          return !this.listoffiletypes.includes(
            "." + cleanNameExt[cleanNameExt.length - 1]
          );
        });

        //console.log(checkFilesExtension);
        //console.log(this.$refs.mltiplefiles.files);
        //console.log(this.updateFileList);

        //Dynamic Create Progress Bar
        if (
          Object.values(this.updateFileList).length > 0 &&
          checkFilesExtension.length === 0
        ) {
          //Reorder Files by size
          this.updateFileList.forEach((value) => {
            //Only Kilobyte
            if (value.size <= 1024) {
              this.reorderFileList.unshift(value);
            }

            //Between KiloByte and Megabyte
            if (value.size >= 1024 && value.size <= 1048576) {
              this.reorderFileList.unshift(value);
            }

            //Between KiloByte and Gigabyte
            if (value.size >= 1048576 && value.size <= this.gigabyte) {
              this.reorderFileList.push(value);
            }

            //When the file is bigger then 2GB
            if (value.size >= this.gigabyte) {
              this.fileSizeAccept = false;
            }
          });

          //console.log(this.reorderFileList);

          //Check Filesize
          if (this.fileSizeAccept) {
            //Get the Last Value
            let lastValueFileList = this.reorderFileList[this.updateFileList.length - 1];

            //Get the Last Index
            let lastIndexFileList = this.reorderFileList.indexOf(lastValueFileList);

            //Generate Progress Bar for each file
            this.reorderFileList.forEach((value, index) => {
              //Create Object to add in SendList
              let fileData = {
                name: value.name,
                filesize: value.size,
                fileBase64: this.toBase64(value),
                displayid: this.displayid,
              };

              //Add fileData in SendFilesList
              this.sendFilesList.push(fileData);

              //Progress Bar Dynamic Create
              this.progressBarSequence(
                value,
                value.name,
                value.size,
                lastIndexFileList,
                index
              );
            });

            //console.log(this.sendFilesList);

            //Clean Variable
            this.reorderFileList = [];
          } else {
            //Clean Variable
            this.reorderFileList = [];

            //Show Error Message
            alert("Favor apenas add ficheiros com até 2GB");
          }
        } else {
          //Show Error When the file is not authorization extension
          alert("Only text files and images are supported.");
        }
      }

      return;
    },
    enableClick() {
      //Enable Event Click from Input File
      this.$refs.mltiplefiles.classList.remove("remove_click");

      //this.$refs.mltiplefiles.addEventListener('click', ()=>{});
    },
    disabledCLick() {
      //Remove Event Click from Input File
      this.$refs.mltiplefiles.classList.add("remove_click");
    },
    dragFile(event) {
      //Enable Event Click from Input File

      this.$refs.mltiplefiles.classList.remove("remove_click");
    },
    progressBarSequence(file, filename, filesize, lastIndex, index) {
      var reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onerror = (evt) => {
        this.errorHandler(evt);
      };
      reader.onabort = function (e) {
        //console.log(e);
        //When enabled Abort Filereader
        if (this.abortSequence) {
          reader.abort();
          alert("File read cancelled");
        }
      };
      reader.onloadstart = function (e) {
        //Only for debug
        //console.log("Start Load");
        //console.log(e.timeStamp);
      };
      reader.onprogress = (evt) => {
        //console.log(evt);

        //Only for debug
        //console.log('Size Percent');
        //console.log(evt.timeStamp);

        // evt is an ProgressEvent.
        if (evt.lengthComputable) {
          //Add More Files Boolean Action - False
          this.addMoreFile = false;

          //Percent Calculate
          var percentLoaded = Math.round((evt.loaded / evt.total) * 100);

          if (this.$refs.borderinputfile.classList.contains("active")) {
            this.$refs.borderinputfile.classList.remove("active");
          }

          //Get the ratio
          //var realtivevalue = Math.ceil(evt.total / evt.loaded);

          //=====> DIV BASE
          //Create Append Element - DivBase
          let divBase = document.createElement("div");
          //Set Attribute - DivBase
          divBase.setAttribute("class", "progressbaseelement");
          divBase.setAttribute("data-fileindex", index);

          //=====> DIV Actions
          //Create Append Element - DivActions
          let divAction = document.createElement("div");

          //Set Attribute - divAction
          divAction.setAttribute("class", "order-1 progress-content div-action");

          //Append Element - divAction
          divBase.appendChild(divAction);

          //Create file icon
          const spanIcon = document.createElement("span");
          spanIcon.innerText = "description";
          spanIcon.setAttribute("class", "material-icons-round");

          //Append SPAN-icon - divAction
          divAction.appendChild(spanIcon);

          //=====> DIV Data
          //Create Append Element - DivData
          let divStructure = document.createElement("div");
          //Set Attribute - DivData
          divStructure.setAttribute("class", "data-structure");

          //Append Element - divData in DivAction
          divAction.appendChild(divStructure);

          //=====> DIV Data
          //Create Append Element - DivData
          let divData = document.createElement("div");
          //Set Attribute - DivData
          divData.setAttribute("class", "divdata");

          //Append Element - divData in DivAction
          divStructure.appendChild(divData);

          //Create Append Element - FileName
          let span = document.createElement("span");
          //Create a span for the icon
          span.setAttribute("class", "material-icons-round");
          span.innerText = "description";

          //Set Attribute - Span
          span.setAttribute("class", "order-1 body01");
          //Add Content
          span.innerText = filename;

          //Create Append Element - FileName
          let spanTwo = document.createElement("span");
          //Set Attribute - Span
          spanTwo.setAttribute("class", "order-2 caption2");
          //Add Content
          spanTwo.innerText = filesize + " bytes";

          //Append First Element in divData
          divData.appendChild(span);

          //Append Second Element in divData
          divData.appendChild(spanTwo);

          //====> DIV Button
          //Create Append Element - DivButton
          let divButton = document.createElement("div");
          //Set Attribute - DivButton
          divButton.setAttribute("class", "divbutton");

          //Create Append Element - Button
          let button = document.createElement("button");
          //Set Attribute - Span
          button.setAttribute("class", "removefile");
          //button.innerText = "Delete";
          //Append Element in divButton
          divButton.appendChild(button);

          //Create span
          let iconTrash = document.createElement("span");
          iconTrash.setAttribute("class", "material-icons-round ");
          iconTrash.innerText = "close";

          //create Span
          let percentProgressBarText = document.createElement("span");
          percentProgressBarText.setAttribute("class", "progressbar--text");
          percentProgressBarText.innerText = `${this.fileprogressbartime} %`;
          button.appendChild(iconTrash);

          //button.appendChild(iconTrash);

          divButton.appendChild(percentProgressBarText);
          //Append Element - divButton in DivAction
          divStructure.appendChild(divButton);

          //=====> DIV Progress
          //Create Append Element - Base Progress Div
          let div = document.createElement("div");
          //Set Attribute - Base Progress Div
          div.setAttribute("class", "progress displayprogress mt-2 order-2");
          div.setAttribute("data-fileindex", index);

          //Append Second Element in DivBase
          divBase.appendChild(div);

          //=====> DIV Progress Bar
          //Create Second Element - Progress Bar
          let progressDiv = document.createElement("div");
          //Set Attribute - Base Progress Div
          progressDiv.setAttribute("class", "progress-bar");
          progressDiv.setAttribute("role", "progressbar");
          progressDiv.setAttribute("aria-valuenow", `${this.fileprogressbartime}`);
          progressDiv.setAttribute("aria-valuemin", "0");
          progressDiv.setAttribute("aria-valuemax", "100");

          //Append ProgressBar in Progress Div
          div.appendChild(progressDiv);

          //Append Progress Div in Progress Base
          document.querySelector(".progressbase").appendChild(divBase);

          //Verify Value for sum
          let verifyValue = percentLoaded !== 0 ? percentLoaded : 15;

          // Increase the progress bar width.
          let percentProgressBar = 0;

          //Only for debug
          /* console.log('Percent Progress Default');
                        console.log(verifyValue);
                        console.log(percentLoaded); */

          //Set Interval
          setInterval(() => {
            //Sum progress bar value
            percentProgressBar += verifyValue;

            //Calculate value closed to 100
            if (percentProgressBar <= 100) {
              //Set Progress Bar Value
              this.fileprogressbartime = percentProgressBar;

              //Add Dynamic Width for Progress Bar
              progressDiv.style.width =
                this.fileprogressbartime >= 70 ? "100%" : this.fileprogressbartime + "%";

              //Set Dynamic Value
              progressDiv.setAttribute("aria-valuenow", this.fileprogressbartime);

              //Only for Debug
              console.log("Less than 100");
              console.log(percentProgressBar);

              //Check total width from Progress Bar - Last Progress Bar
              if (progressDiv.style.width === "100%") {
                //Change Background Color
                //progressDiv.style.backgroundColor =this.progressBarBackgroundColor;
                divBase.classList.add("active");
                iconTrash.innerText = "delete";
                progressDiv.style.display = "none";
                div.style.display = "none";
                divBase.style.border = "none";
                //Verify when is the lastIndex and enable addMoreFiles
                if (index === lastIndex) {
                  console.log("Trigger Last Index");
                  console.log(lastIndex);

                  //Add More Files Boolean Action - True
                  this.addMoreFile = true;

                  return;
                }
              }

              return;
            }
          }, this.timeDefaultProgressBar);

          //When the PercentProgressBar is already 100
          if (verifyValue >= 100) {
            //remove the progressDiv when at 100%
            div.style.display = "none";
            //Add Dynamic Width for Progress Bar
            //progressDiv.style.width = "100%";

            //Change Background Color
            //progressDiv.style.backgroundColor = this.progressBarBackgroundColor;

            //Check Element
            /*  if(allprogress.length > 0){

                                allprogress.forEach((value) => {

                                    //Remove Progress Bar Element
                                    value.remove();

                                });

                                //Append Element in divButton
                                divButton.appendChild(button);

                            } */

            //Verify when is the lastIndex and enable addMoreFiles
            if (index === lastIndex) {
              console.log("Trigger Last Index");
              console.log(lastIndex);

              //Add More Files Boolean Action - True
              this.addMoreFile = true;

              return;
            }
          }
        }
      };
      reader.onloadend = function (e) {
        //Only for debug
        console.log("End Load");
        console.log(e.timeStamp);
      };
    },
    removeActionSequence(event) {
      //Set Element
      let element = event.target;

      //Check when element is a button
      if (element.nodeName === "SPAN") {
        //Get Parent Progress Bar Element
        let parent =
          element.parentElement.parentElement.parentElement.parentElement.parentElement;

        console.log(parent);

        //List All Progress Element
        let allprogress = [...document.querySelector(".progressbase").children];

        //Get Index
        let index = allprogress.indexOf(parent);

        //Abort FileReader
        this.abortReader();

        //Remove Element from FrontEnd FileList
        allprogress[index].remove();

        //Remove value from SendList
        this.sendFilesList.splice(index, 1);

        //Add More Files Boolean Action - True
        this.addMoreFile = true;

        console.log(this.sendFilesList);

        return;
      }

      return;
    },
    arrayUnique(arr) {
      return [...new Set(arr)];
    },
    abortReader() {
      return (this.abortSequence = true);
    },
    sendData(fileData) {
      //Disabled Button
      this.disabled = true;

      //Resolve Base64 Promise
      fileData.forEach((value) => {
        value.fileBase64
          .then((val64) => {
            let fileDataObj = {
              name: value.name,
              filesize: value.filesize,
              fileBase64: val64,
              displayid: value.displayid,
              namecontrol: fileData[fileData.length - 1].name,
              folderid: this.folderid ? this.folderid : null,
            };

            //Send Data
            let response = this.structureTosendDataParent(
              this.urlfilecreate,
              fileDataObj
            );

            //Get Response from Promisse
            response
              .then((data) => {
                //Check if response exist
                if (data) {
                  //Get only Error Messages from Input
                  if (data.hasOwnProperty("errorMessageFromSystem")) {
                    //Check if the Response not is an Array
                    if (!Array.isArray(data.errorMessageFromSystem)) {
                      //Clean Variable
                      this.sendFilesList = [];

                      //Only Show Message when is the last value
                      if (fileData[fileData.length - 1].name === value.name) {
                        //Show error Message
                        alert(data.errorMessageFromSystem);
                      }

                      //Hide Form Modal
                      this.hideModalParent(".formcreatefiles");

                      //Enabled Button
                      this.disabled = false;

                      return;
                    }
                  }

                  //Get only Success Messages
                  if (data.hasOwnProperty("successMessage")) {
                    //Clean Variable
                    this.sendFilesList = [];

                    //Only Show Message when is the last value
                    if (fileData[fileData.length - 1].name === value.name) {
                      //Set New Display List in store from Vuex
                      store.state.folderandfiles = data.attFolderAndFile;

                     /*    //Get LocalStorage
                        let folderInfo = localStorage.getItem('attfolder');
                        
                        //Check Data
                        let attData = folderInfo ? JSON.parse(folderInfo) : {attfolder: []};

                        //Only set File Id
                        let fileIdList = [];

                        //Get FileId
                        data.attFolderAndFile.forEach((value) => {

                            //Only Push File Ids
                            fileIdList.push(value.id);

                        });

                        //Att Folder File List
                        attData.attfolder = fileIdList;

                        //Save in LocalStorage the att file_list by folder
                        localStorage.setItem('attfolder', JSON.stringify(attData)); */

                        //Send Sucess Message
                        //alert(data.successMessage);
                    }

                    //Hide Box Action
                    //document
                    //    .querySelector(".base-action-new")
                    //    .classList.remove("active");
                    //this.showActionBox();

                    //Hide Form Modal
                    this.hideModalParent(".formcreatefiles");

                    //Enabled Button
                    this.disabled = false;

                    //List All Progress Element
                    let allprogress = [
                      ...document.querySelector(".progressbase").children,
                    ];

                    //Remove all fileList in Front
                    allprogress.forEach((valelem) => {
                      valelem.remove();
                    });

                    return;
                  }
                }
              })
              .catch((error) => {
                //Enabled Button
                this.disabled = false;

                throw new Error(error);
              });
          })
          .catch((error) => {
            throw new Error(error);
          });
      });
    },
    errorHandler(evt) {
      switch (evt.target.error.code) {
        case evt.target.error.NOT_FOUND_ERR:
          alert("File Not Found!");
          break;
        case evt.target.error.NOT_READABLE_ERR:
          alert("File is not readable");
          break;
        case evt.target.error.ABORT_ERR:
          break; // noop
        default:
          alert("An error occurred reading this file.");
      }
    },
  },
  beforeMount() {
    //Set List of Authorizated Files
    this.setListOfAuthFiles = this.listoffiletypes.join(",");
  },
  mounted() {
    //Remove Event Click from Input File
    this.$refs.mltiplefiles.classList.add("remove_click");
  },
  computed: {},
  watch: {},
};
</script>

<style lang="scss" scoped>

  @import "./uploadFiles.scss";

</style>
