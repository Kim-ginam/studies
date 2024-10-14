# Redis 세션
## api.properties 설정
```
redis.hostname=192.168.49.2
redis.port=30801
redis.password=password
```
- context-commom.xml
```xml
<bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <!-- <property name="maxTotal" value="${redis.pool.maxTotal}"/>
        <property name="maxIdle" value="${redis.pool.maxIdle}"/>
        <property name="minIdle" value="${redis.pool.minIdle}"/> -->
    </bean>
	
	<bean id="jedisFactory"	class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
		<property name="hostName" value="${redis.hostname}" />
		<property name="port" value="${redis.port}" />
		<property name="password" value="${redis.password}" />
		<property name="poolConfig" ref="jedisPoolConfig"/>
	</bean>
	<bean id="redisTemplate" class="org.springframework.data.redis.core.StringRedisTemplate">
		<property name="connectionFactory" ref="jedisFactory" />
	</bean>
```
# Redis 세션 설정 예시
- 로그인 시 자바 코드
``` java
@Autowired
private StringRedisTemplate redisTemplate;

private static String SUCCESS = "SUCCESS";
private static String ERROR = "ERROR";
private static String SESSION_ID = "session_id";
private static String USER_ID = "user_id";
private static String USER_NAME = "user_name";

@RequestMapping(value = "/user/loginUser", method = RequestMethod.POST)	
@ResponseBody
public  Map<String, Object> loginUser(@RequestParam Map<String, String> paramMap
                        , HttpServletRequest request
                        , HttpServletResponse response) throws Exception {
    HttpSession session = request.getSession(); // HttpServletRequest 으로 세션 정보를 가져옴
    String sessionId = (String)session.getAttribute(SESSION_ID); // 세션 id를 가져옴
    ValueOperations<String, String> pos = redisTemplate.opsForValue(); // redisTemplate.opsForValue()는 Spring Data Redis에서 RedisTemplate을 사용할 때 값(Value) 연산을 위한 API를 제공
    // Redis에서 string 타입의 값을 다룰 때 사용
    String userId = paramMap.get("userId");
    
    Map<String, Object> retMap = new HashMap<String, Object>();
    
    retMap.put("result", this.ERROR);
    System.out.println("SESSION_ID : " + sessionId);
    
    if (userId != null) {
        if( sessionId == null || "".equals(sessionId) || !sessionId.equals(pos.get(userId))) { // !sessionId.equals(pos.get(userId)) 현재 세션 아이디와 유저 아이디를 키로 갖는 세션아이디 값과 같은지 비교
            Map<String, Object> result = userService.checkLogin(paramMap);
            Map<String, Object> userMap = (Map<String,Object>) result.get("resultData");
            String newSessionId = this.getUniqueSessionId();
            
            if(SUCCESS.equals(result.get("result"))) {
                pos.set(userId, newSessionId); // 레디스에 유저아이디를 키로 한 세션 아이디 저장
                session.setAttribute(SESSION_ID, newSessionId);
                session.setAttribute(USER_ID, userId);
                session.setAttribute(USER_NAME, (String)userMap.get("userName"));
            }
            
            retMap.put("result", result.get("result"));
            retMap.put("resultData", result.get("resultData"));
            retMap.put("errMsg", result.get("errMsg"));
        } else {
            retMap.put("result", this.ERROR);
            retMap.put("errMsg", "현재 로그인 중입니다.");
            
        }
    }
    return retMap;
}
```