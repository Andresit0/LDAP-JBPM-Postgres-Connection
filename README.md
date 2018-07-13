La instalación descrita es para tener el logeo que se realiza en LDAP interconectada con la que se realiza con JBPM para ello es necesario tener configurado LDAP con postgres y JBPM con postgres, estas instalaciones se las puede ver en el pdf adjunto llamado "manual_de_configuracion".
Entonces una vez realizado los pasos del "manual de configuración" es necesario manipular la base de datos del LDAP en lo que corresponde a usuario y contraseña para ello en nuestro caso se ha decidido insertar el usuario y contraseña del JBPM en la tabla email del LDAP ya que esta contiene con tiene los usuarios(emails) con los que se logearan por medio de LDAP y JBPM. 
Entonces para ello en PgAdmin4:

1) Añadir una nueva columna a mails en la cual se insertaran los datos correspondientes al password y username del txt de jbpm

alter table mails add column jbpm_user_password varchar;

2) Crear trigger para que cuando guarde con LDAP nuevos usuarios guarde automaticamente la clave en el formato que necesita jbpm. Este trigger esta hecho para que primero se cree un mail y luego se puedan insertar claves, es importante esperar un momento luego de insertar el mail en el apache directory studio antes de insertar la clave caso contrario la misma desaparece

--Creacion de la función que utilizará el trigger 

	create or replace function jbpmPassword() returns trigger as
	$$
	declare
	userpassword varchar;
	begin
   	 select concat_ws('=',mail,New.password ) into userpassword from mails,persons 
    	where mails.pers_id = Old.id and persons.id = Old.id;
    	update mails set jbpm_user_password = userpassword where pers_id = Old.id;
    	copy public.mails (jbpm_user_password) TO '/home/andresito/Instalacion jBPM/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/user.properties' CSV;
    	return NEW;
	end;
	$$
	Language plpgsql;

--Creación del trigger que se ejecutará antes de hacer un UPDATE

	create trigger jbpmUserPassword before update on persons
	for each row execute procedure jbpmPassword();

Nota en caso de haber seguido lo correspondiente al GithHub https://github.com/Andresit0/LDAP-Postgres-Connection que indica el "manual_de_configuracion", con solo copiar, pegar y correr el contenido del archivo "funciones_y_triggers" cambiando la dirección 
/home/andresito/Instalacion jBPM/jbpm-installer/wildfly-8.1.0.Final/standalone/configuration/user.properties por la que contenga el archivo "user.properties" ubicado en la dirección de su JBPM ya estará finalizado la configuración de password y usuarios entre LDAP y JBPM.
