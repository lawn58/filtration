# filtration

       Фильтрация трафика.


   Чтобы проверить задание, нужно чтобы на хостовой машине был свободен порт 8080.
   Необходимо чтобы стандартный ip-адрес eth0 был (по умолчанию public в vagrant) - 10.0.2.15.

   Проверка knocking port.

                       vagrant ssh centralRouter

                       for x in 8881 7777 9991; do sudo nmap -Pn --host_timeout 100 --max-retries 0 -p $x 192.168.255.1; done

                       ssh vagrant@192.168.255.1 [пароль: vagrant]



  Запустить nginx на centralServer, пробросить 80й порт на inetRouter2 8080
  
                        При запуске Vagrantfile, будет доступна начальная страница nginx на хосте, на порту 8080 
