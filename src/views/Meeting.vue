<template>
  <div>
    <button @click="createMeeting">开启会议</button>

    <div id="videos">
      <video id="localVideo" autoplay muted playsinline></video>

      <video v-for="item in remoteVideoList" autoplay playsinline :key="item.id" :id="item.id"></video>
    </div>
  </div>
</template>

<script>
import { io }  from 'socket.io-client';

export default {
  name: 'Home',

  data() {
    return {
      socket: null,
      socketId: null,

      mediaConstraints: {
        audio: false,
        video: true
        // video: false
      },

      room: 'foo',
      meetingRoom: null,
      localStream: null,

      remoteVideoList: [],
      remotePeerList: new Map()
    }
  },

  methods: {
    createMeeting() {
      this.socket.emit('join meeting', this.room);
    },

    handleJoinedMeeting() {
      navigator.mediaDevices.getUserMedia(this.mediaConstraints)
      .then(stream => {
        console.log('Adding local stream.');
        document.getElementById('localVideo').srcObject = stream;
        this.localStream = stream;
      })
    },

    handleNewMemberJoinMeeting(member) {
      this.remoteVideoList.push({
        id: member.socketId
      });

      const peerConnection = this.createPeerConnection(member);
      this.remotePeerList.set(member.socketId, peerConnection);
      this.localStream.getTracks().forEach(track => peerConnection.addTrack(track, this.localStream));
    },

    createPeerConnection(member) {
      const peerConnection = new RTCPeerConnection({
        iceServers: [
          {
            urls: 'stun:stun.stunprotocol.org'
          }
        ]
      });

      peerConnection.onicecandidate = event => {
        if (event.candidate) {
          this.sendMessage({
            type: 'candidate',
            target: member.target,
            source: this.socketId,
            candidate: event.candidate
          })
        }
      }

      peerConnection.ontrack = event => {
        console.log('*** Track event');
        console.log(member);
        const id = member.socketId || member.target;
        document.getElementById(id).srcObject = event.streams[0];
      }

      peerConnection.onnegotiationneeded = () => {
        peerConnection.createOffer().then(offer => {
          return peerConnection.setLocalDescription(offer);
        })
        .then(() => {
          if (member.type != 'offer') {
            this.sendMessage({
              target: member.socketId,
              source: this.socketId,
              type: 'offer',
              sdp: peerConnection.localDescription
            })
          }
        })
      }

      return peerConnection;
    },

    handleVideoOfferMsg(message) {
      const currentMessage = {
        sdp: message.sdp,
        type: message.type,
        source: this.socket,
        target: message.source
      };

      const peerConnection = this.createPeerConnection(currentMessage);
      this.remoteVideoList.push({
        id: currentMessage.target
      })
      this.remotePeerList.set(currentMessage.target, peerConnection);

      const desc = new RTCSessionDescription(currentMessage.sdp);
      peerConnection.setRemoteDescription(desc)
      .then(() => {
        return navigator.mediaDevices.getUserMedia(this.mediaConstraints);
      })
      .then(stream => {
        this.localStream = stream;
        document.getElementById('localVideo').srcObject = stream;
        stream.getTracks().forEach(track => peerConnection.addTrack(track, stream));
      })
      .then(() => {
        return peerConnection.createAnswer();
      })
      .then(answer => {
        return peerConnection.setLocalDescription(answer);
      })
      .then(() => {
        const msg = {
          target: currentMessage.target,
          source: this.socketId,
          type: 'answer',
          sdp: peerConnection.localDescription
        }

        this.sendMessage(msg);
      })
    },

    handleVideoAnswerMsg(message) {
      const peerConnection = this.remotePeerList.get(message.source);

      const desc = new RTCSessionDescription(message.sdp);
      peerConnection.setRemoteDescription(desc);
    },

    sendMessage(message) {
      console.log('Client sending message: ', message);
      this.socket.emit('message', message);
    },
  },

  created() {

  },

  mounted() {
    this.socket = io('https://192.168.124.7:3000');

    if (this.room !== '') {
      this.socket.emit('join room', this.room);
      console.log('Attempted to create or  join room', this.room);
    }

    this.socket.on('created', room => {
      console.log('Created room ' + room);
      this.isInitiator = true;
    });

    this.socket.on('join', room => {
      console.log('Another peer made a request to join room ' + room);
      console.log('This peer is the initiator of room ' + room + '!');
    });

    this.socket.on('joined', (room, id) => {
      console.log('joined: ' + room);
      console.log('socket id: ' + id);
      this.socketId = id;
    });


    this.socket.on('joined meeting', meetingRoom => {
      console.log(`joined meeting ${meetingRoom}`);
      this.meetingRoom = meetingRoom;
      this.handleJoinedMeeting();
    })

    this.socket.on('new meeting', () => {
      alert('new meeting!');
    })

    this.socket.on('new meeting member', member => {
      this.handleNewMemberJoinMeeting(member);
    })

    this.socket.on('send offer', this.doCall);

    this.socket.on('message', message => {
      console.log('Client received message:', message);
      
      if (message.type === 'offer') {
        this.handleVideoOfferMsg(message);
      } else if (message.type === 'answer') {
        console.log('get answer');
        this.handleVideoAnswerMsg(message);
      } else if (message.type === 'candidate') {
        const candidate = new RTCIceCandidate(message.candidate);
        const peerConnection = this.remotePeerList.get(message.source);
        peerConnection.addIceCandidate(candidate)
      } else if (message === 'bye') {
        this.handleRemoteHangup();
      }
    });
  }
}
</script>