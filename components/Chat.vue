<template>
  <div class="container">
	  <div class="row">
	    <div class="col-md-12">
	      <div class="container" ref="details">
          <div class="container" v-if="showName" id="roomjoin">
				    <div class="row">
					    <div class="col-md-12" id="controls">
						    <div class="input-group margin-bottom-md hide" id="registernow">
						      <span class="input-group-btn">
								    <button class="btn btn-success" autocomplete="off" id="register" @click="setUser">{{caption}}</button>
							    </span>
						    </div>
					    </div>
				    </div>
			    </div>
        </div>
	      <div class="container" v-if="showControls" id="room">
	        <div class="row">
		        <div class="col-md-4">
			        <div class="panel panel-default">
			          <div class="panel-heading">
			            <h3 class="panel-title">Participants <span class="label label-info"></span></h3>
			          </div>
			          <div class="panel-body"> 
				          <ul id="list" class="list-group">
						        <li class="list-group-item" v-for="item in participants">
                      {{item}}
                    </li>
			            </ul>
			          </div>
		          </div>
		        </div>
		        <div class="col-md-8">
			        <div class="panel panel-default">
			          <div class="panel-heading">
			            <h3 class="panel-title">Support Chat</h3>
			          </div>
			          <div class="panel-body relative" style="overflow-x: auto;" ref="chatroom">
						      <div v-if="showChat" v-for="item in messages">
							      <p>{{item}}</p>
						      </div>
					      </div>
			          <div class="panel-footer">
				          <div class="input-group margin-bottom-sm">
				            <input class="form-control" type="text" placeholder="Write a chatroom message" v-model="message" id="datasend">
                    <div class="btn-group" role="group" aria-label="Basic example">
                      <button class="btn btn-success" @click="sendData">Send Message</button>
                      <button Class="btn btn-small" @click="destroyJanus">Exit Chat</button>
                    </div>
			            </div>
			          </div>
		          </div>
	          </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import bFormInput from 'bootstrap-vue/es/components/form-input/form-input';
import adapter from 'webrtc-adapter';
import Janus from '@/assets/janus.es';
import moment from 'moment';
import uuidv4 from 'uuid/v4';

export default {
  props: {
    username: {
      String,
      default: 'oops',
    },
    technician: {
      type: Boolean,
      default: false,
    },
    customerid: {
      Number,
      default: 4567,
    },
    caption: {
      String,
      default: 'request support',
    },
  },

  created() {
    this.initJanus();
  },

  mounted() {
    Janus.log('chat component mounted');
  },

  data() {
    return {
      showControls: false,
      showIntro: true,
      showName: false,
      showChat: false,
      janus: null,
      server: 'ws://127.0.0.1/ws',
      participants: [],
      from: '',
      message: '',
      messages: [],
      transactions: [],
      transaction: uuidv4(),
      textroom: null,
    };
  },

  methods: {
    initJanus() {
      Janus.init({
        debug: 'all',
        dependencies: Janus.useDefaultDependencies({
          adapter,
        }),
        callback: () => {
          this.startJanus();
        },
      });
    },

    startJanus() {
      // Create session
      this.janus = new Janus({
        iceServers: [{ url: this.iceServer, username: this.iceUser, credential: this.icePass }],
        server: this.server,
        success: () => {
          // Attach to text room plugin
          this.janus.attach({
            plugin: 'janus.plugin.textroom',
            opaqueId: this.cutomerid,
            success: (pluginHandle) => {
              this.textroom = pluginHandle;
              Janus.log('-------Attached to Janus-------');
              // Janus.log(`Plugin= ${this.textroom.getPlugin()} | ID= ${this.textroom.getId()}`);

              // Setup the DataChannel
              const body = { request: 'setup' };
              // Janus.log(`Sending message ( ${JSON.stringify(body)} )`);
              this.textroom.send({ message: body });
            },
            error: (error) => {
              Janus.log(`Error attaching plugin... ${error}`);
            },
            webrtcState: (on) => {
              Janus.log(on);
              this.showName = true;
            },
            onmessage: (msg, jsep) => {
              // Answer with a peer description in jsep from janus
              if (jsep !== undefined && jsep !== null) {
                Janus.debug(' ::: Got a message :::');
                // Janus.log(JSON.stringify(msg));
                const myJsep = jsep;
                Janus.log('got answer');
                // Janus.log(`jsep is ${JSON.stringify(myJsep)}`);
                this.textroom.createAnswer({
                  jsep: myJsep,
                  media: {
                    audio: false,
                    video: false,
                    data: true,
                  }, // We only use datachannels
                  success: (answerJsep) => {
                    Janus.debug('Got SDP!');
                    const body = { request: 'setup' };
                    this.textroom.send({ message: body, jsep: answerJsep });
                  },
                  error: (error) => {
                    Janus.log('Darn it an error again');
                    Janus.error(`WebRTC error: ${error}`);
                    Janus.log(`WebRTC error... ${JSON.stringify(error)}`);
                  },
                });
              }
            },
            ondataopen: (data) => {
              // Used to prompt for a display name to join the default room
              if (data !== undefined && data !== null) {
                Janus.log('The DataChannel is available!');
                // Janus.log(data);
              }
            },
            ondata: (data) => {
              Janus.debug(`We got data from the DataChannel! ${data}`);
              const json = JSON.parse(data);
              // this.transaction = json.transaction;

              if (this.transactions[this.transaction]) {
                // Someone was waiting for this
                this.transactions[this.transaction](json);
                delete this.transactions[this.transaction];
                return;
              }

              const what = json.textroom;

              if (what === 'message') {
                let msg = json.text;
                msg = msg.replace(new RegExp('<', 'g'), '&lt');
                msg = msg.replace(new RegExp('>', 'g'), '&gt');
                const from = json.from;
                Janus.log(from);
                const dateString = moment().format('HH:mm');
                this.showPublicChat = true;
                this.messages.push(`${dateString} - ${from} - ${msg}`);
              } else if (what === 'join') {
                const user = json.username;
                this.participants.push(user);
              } else if (what === 'success') {
                // Janus.log(`Success json: ${JSON.stringify(json.participants[0].username)}`);
                if (json.participants !== null && json.participants !== undefined) {
                  json.participants.forEach((item) => {
                    Janus.log(`item is ${JSON.stringify(item.username)}`);
                    if (this.participants.includes(item.username) === false) {
                      this.participants.push(item.username);
                    }

                    if (this.participants.includes(this.username) === false) {
                      this.participants.push(this.username);
                    }
                  });
                }
              } else if (what === 'leave') {
                // Somebody left
                this.participants = this.participants.filter(user => user !== json.username);
              } else if (what === 'destroyed') {
                if (json.room !== this.myroom) {
                  return;
                }

                // Room was destroyed, goodbye!
                Janus.warn('The room has been destroyed!');
                Janus.log('The room has been destroyed', () => {
                  window.location.reload();
                });
              }
            },
            oncleanup: () => {
              Janus.log(' ::: Got a cleanup notification :::');
            },
          });
        },
        error: (error) => {
          Janus.error(error);
          Janus.log(error, () => {
            this.resetUI();
          });
        },
        destroyed: () => {
          this.resetUI();
        },
      });
    },

    destroyJanus() {
      this.janus.destroy();
    },

    registerUsername() {
      Janus.log('clicked register');

      const create = {
        textroom: 'create',
        transaction: this.transaction,
        room: this.customerid,
        username: this.username,
        display: this.username,
      };

      const join = {
        textroom: 'join',
        transaction: this.transaction,
        room: this.customerid,
        username: this.username,
        display: this.username,
      };

      this.transactions.push(this.transaction);

      this.transactions[this.transaction] = (response) => {
        // Janus.log(`response is ${JSON.stringify(response)}`);
        if (response.textroom === 'error') {
          // Something went wrong
          if (response.error_code === 417) {
            // This is a "no such room" error: give a more meaningful description
            Janus.log(`Apparently room ${this.customerid} does not exist`);
          } else if (response.error_code === 420) {
            // user name is taken
            this.destroyJanus();
          } else {
            Janus.log(response.error);
          }
        }
        // We're in
        // Janus.log(response);
        this.showIntro = false;
        this.showName = false;
        this.showControls = true;
        this.showChat = true;
      };

      if (this.technician === false) {
        this.textroom.data({
          text: JSON.stringify(create),
          error: (reason) => {
            Janus.log(reason);
          },
        });
      }

      this.textroom.data({
        text: JSON.stringify(join),
        error: (reason) => {
          Janus.log(reason);
        },
      });
    },

    sendData() {
      const message = {
        textroom: 'message',
        transaction: this.transaction,
        room: this.customerid,
        text: this.message,
      };

      this.textroom.data({
        text: JSON.stringify(message),
        error: (reason) => {
          Janus.log(reason);
        },
        success: () => {
          this.message = '';
        },
      });
    },

    resetUI() {
      this.initJanus();
      this.showControls = false;
      this.showIntro = true;
      this.showName = false;
      this.showChat = false;
      this.participants = [];
      this.messages = [];
      this.transactions = [];
    },

    setUser() {
      this.registerUsername();
    },

    components: {
      bFormInput,
      adapter,
      Janus,
      moment,
      uuidv4,
    },
  },
};
</script>

<style scoped>
.nullpad {
  padding: 0;
}

.btn-xs {
  border-radius: 0.15em;
  padding: 0.2rem 0.4875rem;
}

.tooltip > .tooltip-inner {
  font-size: 11px;
}

.wrapper {
  width: 100%;
  color: #eceff1;
  display: grid;
  grid-template-columns: 1fr 1fr 1fr 1fr 2fr;
  grid-template-rows: 240px 240px 240px 240px;
}

.wrapper img {
  width: 100%;
  height: 240px;
}

.badge-top-left {
  position: absolute;
  width: 0;
  height: 0;
  border-top: 40px solid #a1d884;
  border-right: 40px solid transparent;
}

.btn-secondary {
  background-color: rgba(236, 239, 241, 0.5);
  border: 1px solid rgba(236, 239, 241, 0.5);
}

a.nav-link {
  color: #eceff1;
}

.controls-wrapper {
  position: relative;
}

.controls {
  text-align: center;
}

.btn-link:hover {
  color: #007bff;
}

.btn-link {
  color: #fff;
}

.form-control:focus {
  border-color: #cccccc;
  -webkit-box-shadow: none;
  box-shadow: none;
}

input[type="range"] {
  -webkit-appearance: none;
  width: 100%;
  height: 20px;
  margin-top: -84spx;
}

input.form-control[type="range"],
input.form-control[type="color"] {
  height: 1.5rem;
}

.form-button {
  color: #197bd8;
  margin-top: -6px;
}
</style>
