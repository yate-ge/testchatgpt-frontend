<template>
  <div id="app">
    <div class="chat-container">
      <div class="message" v-for="message in messages" :key="message.id" :class="message.sender">
        <div class="content" :class="message.sender ==='user' ?  'content-user' : 'content-server'">
          {{ message.text }}
        </div>
      </div>
    </div>
    <div class="input-container">
      <textarea ref="textarea" v-model="userInput" @keyup.enter="sendMessage" />
      <button @click="sendMessage">发送</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      userInput: "",
      messages: [],
      messageCounter: 0,
      systemMsg: "",
      historyMsg: [],
    };
  },
  methods: {
    async sendMessage() {
      if (this.userInput.trim() === "") return;

      this.addMessage(this.userInput, "user");
      this.historyMsg.push({
        role: 'user',
        content: this.userInput
      });
      //this.fetchReply(this.userInput);
      this.fetchConversationReply();
      this.userInput = "";


      //this.setSystemMsg();

      
      //resize textarea height
      // this.resizeTextarea({ target: this.$refs.textarea });

      //resize textarea height after 0.5 sce
      setTimeout(() => {
        this.resizeTextarea({ target: this.$refs.textarea });
      }, 100);

    },
    addMessage(text, sender) {
      this.messages.push({
        id: this.messageCounter++,
        text,
        sender,
      });


    },
    async fetchReply(text) {
      try {
        const response = await fetch("http://localhost:3333/stream", {
          method: "POST",
          headers: {
            "Content-Type": "application/json",
          },
          body: text,
        });

        
        const reader = response.body.getReader();
        const decoder = new TextDecoder("utf-8");
        let serverResponse = "";

        const processStream = async ({ done, value }) => {
          if (done) {
            return;
          }

          const chunk = decoder.decode(value, { stream: true });
          serverResponse += chunk;
          const lines = serverResponse.split("\n");

          while (lines.length > 1) {
            const line = lines.shift();
            if (line.startsWith("data: ")) {
              const jsonString = line.substring(6);
              if (jsonString.trim() !== "[DONE]") {
                const data = JSON.parse(jsonString);
                const delta = data.choices[0].delta;
                if (delta && delta.content) {
                  // Append the new content to the last server message
                  const serverMessage = this.messages[this.messages.length - 1];
                  if (serverMessage.sender === "server") {
                    serverMessage.text += delta.content;
                  } else {
                    this.addMessage(delta.content, "server");
                  }
                }
              }
            }
          }

          serverResponse = lines[0];
          return reader.read().then(processStream);
        };

        reader.read().then(processStream);
      } catch (error) {
        console.error("Error fetching reply:", error);
        this.addMessage("Error fetching reply", "server");
      }
    },
    async fetchConversationReply() {
      try {
        console.log('historyMsg', this.historyMsg);
        const stringhistory = JSON.stringify(this.historyMsg);
        console.log('stringhistory', stringhistory);
        const response = await fetch("http://localhost:3333/conversation", {
          method: "POST",
          // mode: 'cors',
          headers: {
            "Content-Type": "application/json",
          },
          body: JSON.stringify({ text: this.historyMsg })
        });

        const reader = response.body.getReader();
        const decoder = new TextDecoder("utf-8");
        let serverResponse = "";

        const processStream = async ({ done, value }) => {
          if (done) {
            return;
          }

          const chunk = decoder.decode(value, { stream: true });
          serverResponse += chunk;
          const lines = serverResponse.split("\n");

          while (lines.length > 1) {
            const line = lines.shift();
            if (line.startsWith("data: ")) {
              const jsonString = line.substring(6);
              if (jsonString.trim() !== "[DONE]") {
                const data = JSON.parse(jsonString);
                const delta = data.choices[0].delta;
                if (delta && delta.content) {
                  // Append the new content to the last server message
                  const serverMessage = this.messages[this.messages.length - 1];
                  if (serverMessage.sender === "server") {
                    serverMessage.text += delta.content;
                  } else {
                    this.addMessage(delta.content, "server");

                    this.historyMsg.push({
                      role: 'assistant',
                      content: delta.content
                    });
                  }
                }
              }
            }
          }

          serverResponse = lines[0];
          return reader.read().then(processStream);
        };

        reader.read().then(processStream);
      } catch (error) {
        console.error("Error fetching reply:", error);
        this.addMessage("Error fetching reply", "server");
      }      
    },
    resizeTextarea(event) {
      event.target.style.height = "auto";
      event.target.style.height = event.target.scrollHeight + "px";
    } , 

    // set system message from  prompt.txt assets file
    async setSystemMsg() {
      const fileUrl = 'assets/prompts.txt';
      fetch(fileUrl)
        .then(response => response.text())
        .then(text => {
          this.systemMsg = text;
          console.log('systemMsg', this.systemMsg);

          this.historyMsg.push({
            role: 'system',
            content : this.systemMsg
          });
        }
      )
      .catch(error => {
        console.error('读取文件失败:', error);
      });
    }
    


  },
  mounted() {
    this.$refs.textarea.addEventListener("input", this.resizeTextarea);
    this.setSystemMsg();
  }
};
</script>


<style>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  text-align: center;
  margin-top: 60px;
}

.chat-container {
  width: 900px;
  height: 90%;
  min-height: 500px;
  border: 1px solid #ccc;
  margin: 0 auto;
  overflow-y: auto;
  padding: 10px;
}

.message {
  display: flex;
  margin-bottom: 10px;
  padding: 5px;
  /* max-width: 80%; */
  text-align: left;
}

.message.user {
  justify-content: flex-end;
  
}

.message.server {
  justify-content: flex-start;
  
}

.content {
  border-radius: 5px;
  padding: 10px;
  width: 80%;
  white-space: pre-wrap;
}

.content-user {
  background-color: #00a1ff;
  color: #fff;
}

.content-server {
  background-color: #f1f1f1;
  color: #333;
}

.input-container {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

input[type="text"] {
  width: 700px;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  margin-right: 10px;
  resize: vertical;
}

textarea {
  width: 700px;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  margin-right: 10px;
  resize: none;
  overflow: hidden;
}

textarea:focus {
  outline: none;
}

button {
  background-color: #0080ff;
  color: #fff;
  border: none;
  border-radius: 5px;
  padding: 10px 20px;
  cursor: pointer;
}
</style>