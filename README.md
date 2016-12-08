# resources
others

private static ScriptEngineManager sem = new ScriptEngineManager();
	//var b64map="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
	
	public static String unserialize(String param){
		long st=System.currentTimeMillis();
	    //此处加个为空判断，符合则原样返回。
	    if(param == null || param.trim().length()==0){
	        return param;
	    }
	    
	    try {
            JSONObject json = JSONObject.parseObject(param);
            long et=System.currentTimeMillis();
            logger.info("Decoding has been completed,it cost "+(et-st)+" ms,type 1.");
            return json.toJSONString();
        } catch (Exception e){
            try {
                JSONArray json = JSONArray.parseArray(param);
                long et=System.currentTimeMillis();
                logger.info("Decoding has been completed,it cost "+(et-st)+" ms,type 2.");
                return json.toJSONString();
            } catch (Exception e1) {
            }
        }
	    ScriptEngine se=null;
	    synchronized (sem) {
	    	se = sem.getEngineByName("js");
		}
        String[] arr = param.split("&");
		//case 2:前台传的序列化的URI 例如 a=1&b=2
	    LinkedHashSet<String> lhs=new LinkedHashSet<String>();
	    StringBuffer sb=new StringBuffer();
		lhs.add("var obj={};");
		for(String t :arr){
			if(t==null || t.matches("^[\\d]\\.[\\d]+=$")){
				//排除 jq 自动添加的 随机数参数.
				continue;
			}
			String k="";
			String v="";
			try{
				k=URLDecoder.decode(t.substring(0, t.indexOf("=")), "UTF-8");
				v=URLDecoder.decode(t.substring(t.indexOf("=")+1, t.length()), "UTF-8");
			}catch (UnsupportedEncodingException e) {
	            logger.info("URI解码失败"+e.getMessage());
	        }
			//前台把&与=替换为[and]和[equals]，在这里替换回来
			v = v.replace("[equals]", "=");
			v = v.replace("[and]", "&");
			
			String[] r = getSection(k);
			extendJSON(lhs,r,v);
		}
		lhs.add("var json=JSON.stringify(obj);");
		for (String t:lhs){
			sb.append(t);
		}
		String script=sb.toString();
	    try {
			se.eval(script);
		} catch (ScriptException e) {
			logger.info("转换失败,执行问题:"+e.getMessage());
		}
	    Object o = se.get("json");
	    long et=System.currentTimeMillis();
        logger.info("Decoding has been completed,it cost "+(et-st)+" ms,type 3.");
		if(o!=null){
			return o.toString();
		}else{
			//参数为空.
			return "";
		}
	}
	
	private static void extendJSON(LinkedHashSet<String> lhs,String[] r,String v){
	    if(r==null || r.length==0){
	    	return;
	    }
	    StringBuffer path=new StringBuffer();
	    path.append("obj");
		for(String t:r){
			String currPath=path.toString();
			String temp=currPath+"="+currPath;
			boolean isArray = t.matches("\\[[\\d]*\\]");
			boolean isKey = t.matches("\\w+");
			boolean isMap = t.matches("\\[[\\w]+\\]");
			if(isArray){
				lhs.add(temp+"||[];");
			}else if(isKey){
				t="['"+t+"']";
				lhs.add(temp+"||{};");
			}else if(isMap){
				t=t.replace("[", "['").replace("]", "']");
				lhs.add(temp+"||{};");
			}
			path.append(t);
		}
		if(path.toString().endsWith("[]")){
			boolean isInt=v.matches("(0.)?[\\d]+");
			if(isInt){
				lhs.add(path.toString().replace("[]", "")+".push("+v+");");
			}else{
				lhs.add(path.toString().replace("[]", "")+".push('"+v+"');");
			}
		}else{
			lhs.add(path+"='"+v+"';");
		}
	}
	
	private static String[] getSection(String param){
		ArrayList<String> arr=new ArrayList<String>();
		StringBuffer sb=new StringBuffer();
		for(int i=0;i<param.length();i++){
			if(param.charAt(i)=='['){
				if(sb.length()>0){
					arr.add(sb.toString());
					sb.setLength(0);
				}
				sb.append(param.charAt(i));
			}else if(param.charAt(i)==']'){
				sb.append(param.charAt(i));
				arr.add(sb.toString());
				sb.setLength(0);
			}else{
				sb.append(param.charAt(i));
			}
		}
		if(sb.length()>0){
			arr.add(sb.toString());
		}
		String []resultType={};
		return arr.toArray(resultType);
	}
