# token
token
```
 <?php
ob_start();
/*date_default_timezone_set("Asia/Calcutta");
$created=date('Y-m-d h:i:s a'); //Returns IST*/ 
require_once("../Rest.inc.php");
class API extends REST {
	public $data="";

	const DB_SERVER  = 'localhost';	
	const DB_USER     = 'root';
	const DB_PASSWORD = '';
	const DB          = 'myaxlpl_courier';


	private $db 	= NULL;
	private $mysqli = NULL;
	public function __construct() {	
		parent::__construct();
		$this->dbConnect();
	}
	/* —————————————————————————————————————————————————————————————————————————————————————
	* Coonect to DB
	* ————————————————————————————————————————————————————————————————————————————————————— */
	public function dbConnect() {
		$this->mysqli = new mysqli(self::DB_SERVER,self::DB_USER,self::DB_PASSWORD,self::DB);
		$this->mysqli->set_charset("utf8");
	}
	/* —————————————————————————————————————————————————————————————————————————————————————
	* Dynmically call the method based on the query string
	* ————————————————————————————————————————————————————————————————————————————————————— */
	public function processApi() {
		$func = strtolower(trim(str_replace("/","",$_REQUEST['x'])));
		if((int)method_exists($this,$func)>0) {
			$this->$func();
		}
		else{
			$this->response('','404');
		}
	}
	########################################################################################
	private function categories(){
		if($this->get_request_method() != "GET"){
			$this->response('',406);
		}
		$response['categories'] = array();

		$headerList = [];
		$headers = apache_request_headers();
		foreach ($headers as $header => $value)
		{
		    $headerList[$header] = $value;
		}
		//print_r($headerList);
		if( $headerList["Content-Type"] ==  "application/json" ) 
		{
			if( isset($headerList["Authorization"]) && !empty($headerList["Authorization"])) 
			{
				$authorization = $headerList["Authorization"];
				$arrAuthorization = explode(",", $authorization);

				$bearer = explode(":", $arrAuthorization[0]);
				$client_id = explode(":", $arrAuthorization[1]);

				$query = "SELECT * from z_access_token where account_id='$client_id[1]' AND access_token='$bearer[1]' LIMIT 1";
				//echo $query;
				$res = $this->mysqli->query($query) or die($this->mysqli->error.__LINE__);
				if($res->num_rows > 0) 
				{
					$query_cat = "SELECT * from product_category where active='1'";
					$resCat = $this->mysqli->query($query_cat) or die($this->mysqli->error.__LINE__);
					if($resCat->num_rows > 0) 
					{
						$arr['data'] = array();
						while( $row = $resCat->fetch_assoc() ) 
						{
							$arr['data'] = $row;
						}
						
						$arr["error"] = false;
				        $arr["code"] = 200;
				        $arr["type"] = "success";
				        $arr["message"] = "Success";

						array_push($response["categories"],$arr);
						$this->response($this->json($response), 200);
					}
					else {
						$arr["error"] = true;
				        $arr["code"] = 404;
				        $arr["type"] = "not found";
				        $arr["message"] = "not founded";

						array_push($response["categories"],$arr);
						$this->response($this->json($response), 404);	
					}
				}
				else {
					$arr["error"] = true;
			        $arr["code"] = 401;
			        $arr["type"] = "Unauthorized";
			        $arr["message"] = "Authentication Failure";

					array_push($response["categories"],$arr);
					$this->response($this->json($response), 401);	
				}
			}
			else 
			{
				$arr["error"] = true;
		        $arr["code"] = 400;
		        $arr["type"] = "bad request";
		        $arr["message"] = "Authorization Information is incorrect";

				array_push($response["categories"],$arr);
				$this->response($this->json($response), 400);
			}
		}
		else 
		{
			$arr["error"] = true;
	        $arr["code"] = 400;
	        $arr["type"] = "bad request";
	        $arr["message"] = "Content Type is not specified or specified incorrectly.Content-Type header must be set to application/json";

			array_push($response["categories"],$arr);
			$this->response($this->json($response), 400);	
		}
	}

	private function auth_token(){
		if($this->get_request_method() != "POST"){
			$this->response('',406);
		}
		$response['auth_token'] = array();

		$headerList = [];
		$headers = apache_request_headers();
		foreach ($headers as $header => $value)
		{
		    $headerList[$header] = $value;
		}
		//print_r($headerList);

		if( $headerList["Content-Type"] ==  "application/json" ) 
		{
			if( isset($headerList["Authorization"]) && !empty($headerList["Authorization"])) 
			{
				$paramData = json_decode(file_get_contents('php://input'), true);
				//echo $paramData["grant_type"];
				if( $paramData["grant_type"] == "client_credentials" ) 
				{
					$authorization = $headerList["Authorization"];
					$arrAuthorization = explode(",", $authorization);

					$client_id = explode(":", $arrAuthorization[0]);
					$client_secret = explode(":", $arrAuthorization[1]);

					$query = "SELECT * from z_customer_list where id='$client_id[1]' AND secret_key='$client_secret[1]' LIMIT 1";
					//echo $query;
					$res = $this->mysqli->query($query) or die($this->mysqli->error.__LINE__);
					if($res->num_rows > 0) 
					{
						$arrayData = array(
										'access_token' => "".bin2hex(random_bytes(16))."",
										'created_at' => "".date('c')."",
								        'expires_in' => "36000",
								        'token_type' => "bearer",
								        'account_id' => "$client_id[1]",
								        'shipment_id' => "0",
									);
				        $columns = implode(", ",array_keys($arrayData));
						//$escaped_values = array_map("'", array_values($arrayData));
						$values  = "'".implode("', '", array_values($arrayData))."'";
						$queryIns = "INSERT INTO z_access_token($columns) VALUES ($values)";
						$resIns = $this->mysqli->query($queryIns) or die($this->mysqli->error.__LINE__);

						array_push($response["auth_token"],$arrayData);	
						$this->response($this->json($response), 200);
					}
					else {
						$arr["error"] = true;
				        $arr["code"] = 401;
				        $arr["type"] = "Unauthorized";
				        $arr["message"] = "Authentication Failure";

						array_push($response["auth_token"],$arr);
						$this->response($this->json($response), 401);	
					}
				}
				else 
				{
					$arr["error"] = true;
			        $arr["code"] = 400;
			        $arr["type"] = "bad request";
			        $arr["message"] = "grant_type is incorrect/absent";

					array_push($response["auth_token"],$arr);
					$this->response($this->json($response), 400);	
				}
			}
			else 
			{
				$arr["error"] = true;
		        $arr["code"] = 400;
		        $arr["type"] = "bad request";
		        $arr["message"] = "The authorization information is missing";

				array_push($response["auth_token"],$arr);
				$this->response($this->json($response), 400);
			}
		}
		else 
		{
			$arr["error"] = true;
	        $arr["code"] = 400;
	        $arr["type"] = "bad request";
	        $arr["message"] = "Content Type is not specified or specified incorrectly.Content-Type header must be set to application/json";

			array_push($response["auth_token"],$arr);
			$this->response($this->json($response), 400);	
		}
	}
	########################################################################################
	
	
	/* —————————————————————————————————————————————————————————————————————————————————————
	* 
	*
	*
	* ######## Encode JSON ########
	*
	*
	* 
	* ————————————————————————————————————————————————————————————————————————————————————— */
	private function json($data){
		if(is_array($data)){
			return json_encode($data);
		}
	}
}
/* —————————————————————————————————————————————————————————————————————————————————————
* Initiiate Library
* ————————————————————————————————————————————————————————————————————————————————————— */
$api = new API;
$api->processApi();	
?>
```
