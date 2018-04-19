Retry handler in spring integration 

<bean class="org.springframework.integration.http.inbound.UriPathHandlerMapping"/>
	<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver"/>
	
	<int-http:inbound-gateway
			id="httpInboundGateway"
			request-channel="httpRequestChannel"
			reply-channel="httpResponseChannel"
			supported-methods="POST"
			path="/xxxx"
			error-channel="callbacksErrorChannel">
	</int-http:inbound-gateway>

	<int:channel id="httpRequestChannel" />	
	<int:channel id="httpResponseChannel" />
	
	<!-- Subscriber to cnpshHttpInboundGateway -->
        <int:chain id="callbackChain" 
        	input-channel="httpRequestChannel"
    	        output-channel="jmsInputChannel">
         	<int:filter expression="payload != null 
    				and payload.getClass().getName().equals('org.springframework.util.LinkedMultiValueMap')	
    				and !payload.isEmpty()"/>
        	<int:transformer method="transform" ref="messageToMapTransformer"/>
    	        <!-- Remove all HTTP header values -->
         	<int:header-filter header-names="*"/>
    </int:chain>

	<!-- JMS OUTBOUND MESSAGING -->
	<int:channel id="jmsInputChannel" />
	
	<!-- JMS Producer -->
	<int-jms:outbound-channel-adapter auto-startup="true" id="jmsOutAdapter" channel="jmsInputChannel" destination-name="CallbackQueue">
 		<int-jms:request-handler-advice-chain>
	        <bean class="org.springframework.integration.handler.advice.RequestHandlerRetryAdvice">
	            <property name="recoveryCallback">
	                <bean class="org.springframework.integration.handler.advice.ErrorMessageSendingRecoverer">
	                    <constructor-arg ref="outboundErrorChannel" />
	                </bean>
	            </property>
	            <property name="retryTemplate" ref="retryTemplate" />
	        </bean>
	    </int-jms:request-handler-advice-chain>
	</int-jms:outbound-channel-adapter>
	
	<bean id="retryTemplate" class="org.springframework.retry.support.RetryTemplate">
		<property name="retryPolicy">
			<bean class="org.springframework.retry.policy.SimpleRetryPolicy">
				<property name="maxAttempts" value="2" />
			</bean>
		</property>
		<property name="backOffPolicy">
			<bean class="org.springframework.retry.backoff.FixedBackOffPolicy">
				<property name="backOffPeriod" value="1000" />
			</bean>
		</property>
	</bean>

 	<int:channel id="outboundErrorChannel"/>
	<int:service-activator input-channel="outboundErrorChannel" ref="outboundErrorLogger" method="logOutboundError"/>

	<int:channel id="callbacksErrorChannel"/>
	<int:chain input-channel="callbacksErrorChannel">
		<int:transformer ref="errorUnwrapper" />
		<int:service-activator ref="errorLogger" method="logError"/>
	</int:chain>
