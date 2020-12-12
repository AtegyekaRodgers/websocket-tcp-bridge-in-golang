# websocket-tcp-bridge-in-go

The purpose of this project is to enable us communicate with a TCP server using a web browser. This is very useful when you need to remotely control an IoT device 
through a Wireless Sensor Network gateway which needs to use strictly TCP protocol to send/receive information to/from a web server.

This 'bridge' server is written in golang, communicating with another TCP server developed in node js, which in turn communiactes to the gateway device (via cellular network using GSM) installed at a weather station in the range of a wireless sensor network for capturing weather information from environment detcting sensors.

#### Code
``` package main

import (
	"flag" 
	"strings" 
	"html/template"
	"log"
	"net"
	"net/http" 
	"github.com/gorilla/websocket"
)

var addr = flag.String("addr", "0.0.0.0:10026", "websocket-tcp bridge service address")

var upgrader = websocket.Upgrader{} // use default options 

type Wc struct {
	Websocket *websocket.Conn
}


func (wscon *Wc) bridge(tcpcon net.Conn, mt int) {
	resBuf := make([]byte, 4096)
	defer wscon.Websocket.Close()
	defer tcpcon.Close()  
	for { 	
		length, err := tcpcon.Read(resBuf)  //read from tcpcon  
		if err != nil {
		    log.Println("readtcp error:", err) 
		} 
		if length > 0 {
			log.Println("RECEIVED from tcp: " + string(resBuf))
			err = wscon.Websocket.WriteMessage(mt, resBuf) //responding to the web client.
			if err != nil {
				log.Println("response Error:", err) 
				break
			}
		} 
	}
}


func createNewBridge(w http.ResponseWriter, r *http.Request) {
	upgrader.CheckOrigin = func(r *http.Request) bool { return true }
	webclient, _ := upgrader.Upgrade(w, r, nil)   
	connectedToTcp := false 
	var tcpcon net.Conn
	
	for {
		log.Println("Waiting for data from websocket client ...")
		datatype, dataFromClient, err := webclient.ReadMessage() 
		if err != nil {
			log.Println("!! Error:", err) 
		} 
		log.Println("Received data-: ",dataFromClient)
		log.Println("datatype: ",datatype," Data in string-: ",string(dataFromClient)) 
		msgtokens := strings.Fields(string(dataFromClient)) //breaking the received string message
		//TODO: do the above only if the message from client is a string and if not, skip above operation. 
		if(msgtokens[0]=="connect"){
		   tcpcon, err = net.Dial("tcp", msgtokens[1]) 
		   if err!=nil {
		   	 log.Println("Error while connecting to TCP-: ",err)
		   }else{
				connectedToTcp = true
		   	 /********************************************************/
		   	 wc := &Wc{Websocket:webclient}      /********************/ 
		     go wc.bridge(tcpcon, datatype)      /*******Bridge*******/
		     /********************************************************/
			}
		}
		if(connectedToTcp){
			if(msgtokens[0]=="disconnect"){  
			   connectedToTcp = false
			}else{
				if(msgtokens[0]=="identity" || msgtokens[0]=="To" || msgtokens[0]=="ToAll" || msgtokens[0]=="register" || msgtokens[0]=="login"){
					log.Printf("Forwarding to tcp socket: %s", dataFromClient)  
					_, err = tcpcon.Write(dataFromClient)  /* submits the data from client to TCP end point */
					if err != nil {
						log.Println("Failed to forward:", err) 
					}
				}
		    } 
		}  
	}
}


func home(w http.ResponseWriter, r *http.Request) {
	homeTemplate.Execute(w, "ws://"+r.Host+"/echo")
}
  

func main() {
	flag.Parse()
	log.SetFlags(0)
	http.HandleFunc("/bridge", createNewBridge)
	http.HandleFunc("/", home)
	log.Println("Websocket-tcp bridge listening on port: 10026") 
	log.Fatal(http.ListenAndServe(*addr, nil)) 
}



 
var homeTemplate = template.Must(template.New("").Parse(`
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">

</head>

<body>
	<div> <h4>You should <br/><h1>Use the React client</h1> <br/>to access this system </h4> </div>
</body>
</html>
`)) 

```

