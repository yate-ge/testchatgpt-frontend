<template>
  <div id="app">
    <div class="chat-container">
      <div class="message" v-for="message in messages" :key="message.id" :class="message.role">
        <div class="content" :class="message.role ==='user' ?  'content-user' : 'content-assistant'">
          {{ message.content }}
        </div>
      </div>
    </div>
    <div class="input-container">
      <textarea ref="textarea" v-model="userInput" />
      <button @click="sendMessage">发送</button>
    </div>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      userInput: "",
      messages: [],
      messageCounter: 0,
      systemMsg: "",
      historyMsg: [],
      currentMsg: ""
    };
  },
  methods: {
    async sendMessage() {
      if (this.userInput.trim() === "") return;

      let sendContent = this.userInput;


      //取回node-red 当前的flow，并判断是否为空
      try {
        const response = await axios.get("http://127.0.0.1:1880/flows");
        const flows = response.data;

        // 检查flows是否为空
        if (flows.length === 0) {
          console.log('当前flow为空');
          
        } else {
          // 检查是否仅有一个tab节点且没有其他节点
          if (flows.length === 1 && flows[0].type === 'tab') {
            console.log('当前flow为空');
          } else {
            console.log('当前flow不为空');

            sendContent = "请按照以下需要对代码进行修改：\n" + this.userInput + "\n" + "代码: \n" + response.data;
          }
        }
      } catch (error) {
        console.error('获取flows失败：', error.message);
        return false;
      }    





      //const sendContent = "请按照以下需要对代码进行修改：\n" + this.userInput + "\n" + "代码: \n" + response.data;

      this.addMessage(this.userInput, "user");
      this.addMessage("正在生成中....", "assistant");
      this.historyMsg.push({
        role: 'user',
        content: sendContent
      });
      //this.fetchReply(this.userInput);
      this.fetchConversationReply();
      this.userInput = "";


      //resize textarea height after 0.5 sce
      setTimeout(() => {
        this.resizeTextarea({ target: this.$refs.textarea });
      }, 100);

    },
    addMessage(content, role) {
      this.messages.push({
        id: this.messageCounter++,
        content,
        role,
      });

    },
    // async fetchReply(text) {
    //   try {
    //     const response = await fetch("http://localhost:3333/stream", {
    //       method: "POST",
    //       headers: {
    //         "Content-Type": "application/json",
    //       },
    //       body: text,
    //     });

        
    //     const reader = response.body.getReader();
    //     const decoder = new TextDecoder("utf-8");
    //     let serverResponse = "";

    //     const processStream = async ({ done, value }) => {
    //       if (done) {
    //         return;
    //       }

    //       const chunk = decoder.decode(value, { stream: true });
    //       serverResponse += chunk;
    //       const lines = serverResponse.split("\n");

    //       while (lines.length > 1) {
    //         const line = lines.shift();
    //         if (line.startsWith("data: ")) {
    //           const jsonString = line.substring(6);
    //           if (jsonString.trim() !== "[DONE]") {
    //             const data = JSON.parse(jsonString);
    //             const delta = data.choices[0].delta;
    //             if (delta && delta.content) {
    //               // Append the new content to the last server message
    //               const serverMessage = this.messages[this.messages.length - 1];
    //               if (serverMessage.sender === "server") {
    //                 serverMessage.text += delta.content;
    //               } else {
    //                 this.addMessage(delta.content, "server");
    //               }
    //             }
    //           }
    //         }
    //       }

    //       serverResponse = lines[0];
    //       return reader.read().then(processStream);
    //     };

    //     reader.read().then(processStream);
    //   } catch (error) {
    //     console.error("Error fetching reply:", error);
    //     this.addMessage("Error fetching reply", "server");
    //   }
    // },
    async fetchConversationReply() {
      try {
        //console.log('historyMsg', this.historyMsg);
        //const stringhistory = JSON.stringify(this.historyMsg);
        //console.log('stringhistory', stringhistory);
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


        // 在处理前显示正在生成中的信息
        //this.addMessage("正在生成中....", "assistant");
        //let serverResponseProcessed = "";

        const processStream = async ({ done, value }) => {
          if (done) {
            //const serverMsg = this.messages[this.messages.length - 1];
            //serverMsg.content = "生成完成";
            this.processResultfromGPT();
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
                  //serverResponseProcessed += delta.content;

                  const lastMsg = this.historyMsg[this.historyMsg.length - 1];
                  console.log('lastMsg', lastMsg.role);
                  if (lastMsg.role === "assistant") {
                    lastMsg.content += delta.content;
                  } else {
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
    async processResultfromGPT() {
      //通过post向“http://127.0.0.1:1880/flows”发起请求，将生成的文本传递给node-red。headers中的“Content-Type”为“application/json”，“Node-RED-Deployment-Type”为“full”。body是messages中的最后一条消息的content。

      if(this.currentMsg.length === 0) {
        this.currentMsg = this.historyMsg[this.historyMsg.length - 1].content;
        console.log('rawdata', this.currentMsg);
      } else {
        this.currentMsg += this.historyMsg[this.historyMsg.length - 1].content;
        console.log('rawdata', this.currentMsg);
      }

      // 对当前msg进行处理，将内容提取出来，如果提取不出来的话，表示内容不完整，再次请求服务器

      try {
        const donedataMatch = this.currentMsg.match(/<done>([\s\S]*?)<\/done>/);
        if (donedataMatch) {
          console.log('已经输出完成内容，rawdata', this.currentMsg);
          // 表示内容完整，进行下一步处理
          //const donedata = donedataMatch[1];

          // 处理flow中的内容
          const flowdata = this.currentMsg.match(/<flow>([\s\S]*?)<\/flow>/)[1];
          console.log('flowdata', flowdata);

          // 处理<describ>中的内容，并在UI上进行反馈
          const describdata = this.currentMsg.match(/<describ>([\s\S]*?)<\/describ>/)[1];
          console.log('describdata', describdata);
          const serverMsg = this.messages[this.messages.length - 1];
          serverMsg.content = "生成完成，我将按照一下内容进行部署：\n" + describdata;

          // 将生成的内容发送给node-red
          const headers = {
            "Content-Type": "application/json",
            "Node-RED-Deployment-Type": "full"
          };
          const response = await axios.post('http://127.0.0.1:1880/flows', flowdata, { headers: headers });

          // 处理响应
          console.log(response);
          this.refreshTargetPage();

          // 清空当前msg
          this.currentMsg = "";

        }else{
          // 内容不完整，再次请求服务器
          //this.fetchConversationReply();
          console.log('内容不完整，再次请求服务器');

          const gotoPrompt = "go on";

          //this.addMessage(gotoPrompt, "user");
          this.historyMsg.push({
            role: 'user',
            content: gotoPrompt
          });
          //this.fetchReply(this.userInput);
          this.fetchConversationReply();
        }

      }catch (error) {
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
    },
    refreshTargetPage() {
      window.postMessage(
        {
          type: 'MY_CUSTOM_EVENT',
          data: {
            message: 'refresh',
          },

        }, '*'


      );


      console.log('refreshTargetPage');
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
  width: 500px;
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

.message.assistant {
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

.content-assistant {
  background-color: #f1f1f1;
  color: #333;
}

.input-container {
  display: flex;
  justify-content: center;
  margin-top: 20px;
}

input[type="text"] {
  width: 400px;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 5px;
  margin-right: 10px;
  resize: vertical;
}

textarea {
  width: 400px;
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