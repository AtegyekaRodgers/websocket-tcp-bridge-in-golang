# websocket-tcp-bridge-in-go

The purpose of this project is to enable us communicate with a TCP server using a web browser. This is very useful when you need to remotely control an IoT device 
through a Wireless Sensor Network gateway which needs to use strictly TCP protocol to send/receive information to/from a web server.

This 'bridge' server is written in golang, communicating with another TCP server developed in node js, which in turn communiactes to the gateway device (via cellular network using GSM) installed at a weather station in the range of a wireless sensor network for capturing weather information from environment detcting sensors.

#### Code
``` 
package main

import (
	"flag"
	"fmt"
	//"bufio"
	"strings"
	//"encoding/json"
	"html/template"
	"log"
	"net"
	"net/http" 
	"github.com/gorilla/websocket"
	"math/rand"
	"strconv"
	"database/sql" 
	_"github.com/go-sql-driver/mysql"
)

type newClient struct {
	SessionId float64 `json:"sessionid"` 
	StartTime float64 `json:"starttime"` 
	StayDuration int64 `json:"stayduration"`  
} 
var chanoToSessionManager = make(chan *newClient) 
var wsEndPoint = ":10026"
var sessionManagerEndPoint = ":10027"
var tcpEndPoint = "wimea.mak.ac.ug:10028"


var addr = flag.String("addr", wsEndPoint, "websocket-tcp bridge service address")

var upgrader = websocket.Upgrader{} // use default options 

type Wc struct {
	Websocket *websocket.Conn
}

func dbconnect() *sql.DB {
	db, err := sql.Open("mysql","root:@tcp(127.0.0.1:3306)/wdrDb")
	if err != nil {  fmt.Println("Failed to connect to the dabtabase"); fmt.Println(err.Error()) 
		return nil
	} 
	return db
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

func startSessionFor(thisClient *newClient) {
	chanoToSessionManager <- thisClient
}

func newConnectHandler(w http.ResponseWriter, r *http.Request) {
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
		if(msgtokens[0]=="login" && (!connectedToTcp)){  
			//read login credentials from second part of the message after space;- 
			//NOTE:login request message expected to look like: "login user:rodgers-pass:Seqr3tp@55w0rd"
			uname := strings.Split((strings.Split(msgtokens[1],"-")[0]),":")[1]
			pword := strings.Split((strings.Split(msgtokens[1],"-")[1]),":")[1]
			//athenticate the client by retrieving credentials from database
			db := dbconnect()
			defer db.Close()
			selecQuery := fmt.Sprintf("SELECT * FROM userstable WHERE username='%s' AND password='%s' ",uname,md5(pword))
			queryResult, err := db.Query(selecQuery)
			if err != nil {  fmt.Println(err.Error())  }  
			defer queryResult.Close() 
			userExists := 0
			for queryResult.Next(){ 
				userExists =1 
				break
			}
			if userExists {
				//generate a session id	
				newSessionId := fmt.Sprintf("CL%s",strconv.Itoa(rand.Intn(100000))) 
				//send session id to client 
				err = webclient.WriteMessage(datatype, newSessionId) //responding to the web client.
				if err != nil {
					log.Println("response Error:", err)  
				}
				//send session id to session manager  
				thisClient := newClient{ SessionId: newSessionId }
				startSessionFor(thisClient)
				//create a new tcp connection and a go routine to serve the connection. 
				//NOTE: defer close statement is inside the go routine, dont write it here. just pass the new tcpcon object.
				tcpcon, er := net.Dial("tcp", tcpEndPoint)
				if er==nil {
					connectedToTcp = true
					/********************************************************/
					wc := &Wc{Websocket:webclient}      /********************/ 
					go wc.bridge(tcpcon,datatype)       /*******Bridge*******/
					/********************************************************/
				} 
			} 
		}
		if(connectedToTcp){
			if(msgtokens[0]=="disconnect"){  
				connectedToTcp = false
				webclient.Close()
			}else{
				log.Printf("Forwarding data to tcp server ...")  
				_, err = tcpcon.Write(dataFromClient)
				if err != nil {
					log.Println("Failed to forward:", err) 
				}else{ log.Println(" Ok . ") }
			}
		}  
	}
}

func reconnectHandler(w http.ResponseWriter, r *http.Request) {
	//read the http headers for a session id/key
	sessionid := r.ReadHeader("Session-Id")
	if sessionid=="" {return }
	//submit the session key to the session manager
	tcpconMan, er:= net.Dial("tcp",sessionManagerEndPoint)
	defer tcpconMan.Close()
	if er==nil { 
		_, err = tcpconMan.Write(sessionid)
		if err != nil {
			log.Println("Failed to Consult session manager via tcp :", err) 
		}else{ 
			intBuf := make([]byte, 8)
			//read manager's response
			_, err := tcpconMan.Read(intBuf)
			ok := intBuf //we expect the session manager to respond with an integer; 1 for ok , 0 for not ok
			//if session manager's response is 'OK' , do the following and then terminate:
			if(ok){
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
					if !connectedToTcp {   
						//connect to the tcp server
						tcpcon, er := net.Dial("tcp", tcpEndPoint)
						if er==nil {
							connectedToTcp = true
							/********************************************************/
							wc := &Wc{Websocket:webclient}      /********************/ 
							go wc.bridge(tcpcon,datatype)       /*******Bridge*******/
							/********************************************************/
						}  
					}
					if(connectedToTcp){
						if(msgtokens[0]=="disconnect"){  
							connectedToTcp = false
							webclient.Close()
						}else{
							log.Printf("Forwarding data to tcp server ...")  
							_, err = tcpcon.Write(dataFromClient)
							if err != nil {
								log.Println("Failed to forward:", err) 
							}else{ log.Printf(" Ok . \n ") }
						}
					}  
				}
			}
		}
	}  
	
}

func home(w http.ResponseWriter, r *http.Request) {
	homeTemplate.Execute(w, "ws://"+r.Host+"/newcon")
}

func sessionManagerStart(){
	Buf := make([]byte, 4096)
	//create a tcp server, ListenAndServe
	tcpserver, err := net.Listen("tcp", strings.Split(tcpEndPoint,":")[1])
	if err != nil {
		fmt.Println(err) 
	}
	defer tcpserver.Close()
	
	// keep reading a channel for newly connected clients   
	var clients = make(map[*newClient]bool)
	for { 
		select {
			case (var newcon, err = tcpserver.Accept()):  //is there a new connection request? skip if no - (non blocking)
			if err != nil {
				fmt.Println(err) 
			}else
				if err == nil {
					_, err := newcon.Read(Buf)  //read from tcpcon  
					if err != nil {
						log.Println("tcp read error:", err) 
					}else{
						// look for a client with the received session id
						var x = 0
						for client := range clients {  
							if clients.SessionId==string(Buf)  {
								x=1  //if the session id is found, increament x ; otherwise x remains 0
								break
							} 
						}
						_, err = newcon.Write(x) 
						if err != nil {
							log.Println("Session manager failed to respond:", err) 
						}
						newcon.Close() 
					}
					
				}  
			case newClient := <-chanoToSessionManager:  //is there a newly connected client? skip if not - (non blocking)
			//read current timestamp and attach to newClient as start time
			newClient.StartTime = time.Now().Unix() 
			newClient.StayDuration = (time.Now().Unix())+(1*60*60)  //maximum active session time is 1 hr 
			clients[newClient] = true 
		}//end of select
		
		// if any client's session duration elapsed, delete from clients map
		currentTimeStamp := time.Now().Unix() 
		for client := range clients { 
			if currentTimeStamp >= (client.StartTime+client.StayDuration) { 
				if clients[client]==true {
					delete(clients, client)
				}
			}
		}
		
	}   
	
}

func main() {
	flag.Parse()
	log.SetFlags(0)
	http.HandleFunc("/newcon", newConnectHandler)
	http.HandleFunc("/recon", reconnectHandler)
	http.HandleFunc("/", home)
	log.Println("Websocket-tcp bridge listening on port: "+(strings.Split(wsEndPoint,":")[1]) )  
	go sessionManagerStart()
	log.Fatal(http.ListenAndServe(*addr, nil)) 
}




var homeTemplate = template.Must(template.New("").Parse(`
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<script>  
window.addEventListener("load", function(evt) {
var output = document.getElementById("output");
var input = document.getElementById("input");
var ws;
var print = function(message) {
var d = document.createElement("div");
d.innerHTML = message;
output.appendChild(d);
};
document.getElementById("open").onclick = function(evt) {
if (ws) {
	return false;
	}
	ws = new WebSocket("{{.}}");
ws.onopen = function(evt) {
print("OPEN");
}
ws.onclose = function(evt) {
print("CLOSE");
ws = null;
ws = new WebSocket("`+(wsEndPoint)+`/recon"`+`);
}
ws.onmessage = function(evt) {
print("RESPONSE: " + evt.data);
}
ws.onerror = function(evt) {
print("ERROR: " + evt.data);
}
return false;
};
document.getElementById("send").onclick = function(evt) {
if (!ws) {
	return false;
	}
	print("SEND: " + input.value);
ws.send(input.value);
return false;
};
document.getElementById("close").onclick = function(evt) {
if (!ws) {
	return false;
	}
	ws.close();
return false;
};
});
</script>
</head>
<body>
<table>
<tr><td valign="top" width="50%">
<p>Click "Open" to create a connection to the server, 
"Send" to send a message to the server and "Close" to close the connection. 
You can change the message and send multiple times.
<p>
<form>
<button id="open">Open</button>
<button id="close">Close</button>
<p><input id="input" type="text" value="Hello world!">
<button id="send">Send</button>
</form>
</td><td valign="top" width="50%">
<div id="output"></div>
</td></tr></table>
</body>
</html>
`))
```
//NOTE: this code is not yet fully complete, some sections are not yet tested. Eg the session manager, i plan to 
//redesign it to use a database instead.
//However if you follow through the code you can easily understand the whole logic and then improve on those incomlete sections while 
//am also continuing to add more code 
