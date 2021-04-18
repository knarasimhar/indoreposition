# indoreposition

  # C# web api to read the node and wifi data from wifi repeater
  
       
        [System.Web.Http.Route("api/Pipflow/IOTBulkPushAPIS")]
        [System.Web.Http.HttpPost]
        public HttpResponseMessage IOTBulkPushAPIS([FromBody] JToken postData, HttpRequestMessage request)
        {
            if (ClsGeneral.getConfigvalue("FROM_SSOM_URL_INSERT") != "")
                return getHttpResponseMessage(ClsGeneral.DoPostWebreqeust(ClsGeneral.getConfigvalue("FROM_SSOM_URL") + "api/Pipflow/IOTBulkPushAPIS" + ControllerContext.Request.RequestUri.Query.ToString(), JsonConvert.SerializeObject(postData)));

            ClientContext clientContext = new ClientContext(strSiteURL);
            clientContext.Credentials = new NetworkCredential(strUSER, strPWD);

            //Get the list items from list
            SP.List oList = clientContext.Web.Lists.GetByTitle("bulkpushapis");
            ListItemCreationInformation oListItemCreationInformation = new ListItemCreationInformation();


            ListItem oItem = oList.AddItem(oListItemCreationInformation);

            try
            {

                oItem["Title"] = "IOT DATA";
                oItem["status"] = "-9";
                oItem["pushurl"] = postData.ToString();

                string strLog = "";
                //   List<IotDevice> iotDobj = new List<IotDevice>

                double finalDistance = 0, distance, ratio;
                Hashtable _htDivices = new Hashtable();
                List<IotDevice> obj = JsonConvert.DeserializeObject<List<IotDevice>>(postData.ToString());

                foreach (IotDevice sobj in obj)
                {
                    if (sobj.type.ToLower() == "gateway")
                    { strLog = sobj.mac + ",gateway,"; break; }

                }

                foreach (string strDevic in ClsGeneral.getConfigvalue("macs").Split(','))
                {

                    foreach (IotDevice sobj in obj)
                    {

                        if (sobj.mac == strDevic && sobj.ibeaconTxPower != null)
                        {

                            ratio = double.Parse(sobj.rssi) * 1.0 / (double.Parse(sobj.ibeaconTxPower));
                            // var distance = 0;
                            if (ratio < 1.0)
                            {
                                distance = Math.Pow(ratio, 10);
                            }
                            else
                            {
                                distance = (0.89976) * Math.Pow(ratio, 7.7095) + 0.111;
                                // return distance;
                            }
                            // double distance = Math.Pow(10, ((double.Parse(sobj.ibeaconTxPower) - double.Parse(sobj.rssi)) / (10 * 2)));
                            //  strLog += sobj.mac + "," + Math.Round(distance,2).ToString() + ",";
                            if (_htDivices.Contains(strDevic))
                                _htDivices[strDevic] = double.Parse(_htDivices[strDevic].ToString()) + distance;
                            else
                                _htDivices.Add(strDevic, distance);

                            if (_htDivices.Contains(strDevic + "_count"))
                                _htDivices[strDevic + "_count"] = int.Parse(_htDivices[strDevic + "_count"].ToString()) + 1;
                            else
                                _htDivices.Add(strDevic + "_count", 1);
                            // break;
                        }
                    }
                    if (_htDivices.Contains(strDevic))
                        finalDistance = Math.Round(double.Parse(_htDivices[strDevic].ToString()) / int.Parse(_htDivices[strDevic + "_count"].ToString()), 2);
                    strLog += strDevic + "," + finalDistance.ToString() + ",";
                }
                if (ClsGeneral.getConfigvalue("macs") != "")
                    oItem["log"] = strLog;
                else
                    oItem["log"] = postData.ToString();
            }
            catch (Exception ex)
            {
                oItem["pushurl"] = ex.Message;
                // return getErrormessage(ex.Message);
            }

            oItem.Update();
            //clientContext.Load(oItem);
            clientContext.ExecuteQuery();

            return getSuccessmessage("Success");
        }
