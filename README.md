# projeto-urbano

package projeto.model;

import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;

@Entity
@Table(name = "Denuncias")
public class Denuncia {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String titulo;
    private String descricao;
    private String local;
    private String tipo;
    private String status;
    private String caminhoFoto;

    @ManyToOne
    @JoinColumn(name = "usuario_id")
    private Usuario usuario;

    // Construtores (se houver)

    // Getters e setters para todos os campos
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getTitulo() {
        return titulo;
    }

    public void setTitulo(String titulo) {
        this.titulo = titulo;
    }

    public String getDescricao() {
        return descricao;
    }

    public void setDescricao(String descricao) {
        this.descricao = descricao;
    }

    public String getLocal() {
        return local;
    }

    public void setLocal(String local) {
        this.local = local;
    }

    public String getTipo() {
        return tipo;
    }

    public void setTipo(String tipo) {
        this.tipo = tipo;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    public Usuario getUsuario() {
        return usuario;
    }

    public void setUsuario(Usuario usuario) {
        this.usuario = usuario;
    }

    public String getCaminhoFoto() {
        return caminhoFoto;
    }

    public void setCaminhoFoto(String caminhoFoto) {
        this.caminhoFoto = caminhoFoto;
    }
}

package projeto.model;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;
import java.util.List;

@Entity
@Table(name = "usuarios")
public class Usuario {
    

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    @OneToMany(mappedBy = "usuario")
    private List<Denuncia> denuncias;

    // Getters e Setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public List<Denuncia> getDenuncias() {
        return denuncias;
    }

    public void setDenuncias(List<Denuncia> denuncias) {
        this.denuncias = denuncias;
    }
}
package projeto.projetourbano.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;

import projeto.model.Denuncia;
import projeto.model.Usuario;
import projeto.projetourbano.service.DenunciaService;
import projeto.projetourbano.service.UserService;

@Controller
@RequestMapping("/denuncias")
public class DenunciaController {

    @Autowired
    private DenunciaService denunciaService;

    @Autowired
    private UserService userService;

    @GetMapping("/cadastro")
    public String exibirFormularioCadastro(Model model) {
        model.addAttribute("denuncia", new Denuncia());
        return "cadastro_denuncia";
    }

    @GetMapping("/listar")
    public String listarDenuncias(Model model) {
        List<Denuncia> denuncias = denunciaService.listarDenuncias();
        model.addAttribute("listaDenuncias", denuncias);
        return "denuncias/listar";
    }

    @PostMapping("/salvar")
    public String salvarDenuncia(@ModelAttribute("denuncia") Denuncia denuncia,
                                 @RequestParam(value = "foto", required = false) MultipartFile foto) {

        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        String username = authentication.getName();
        Usuario usuario = userService.buscarPorUsername(username);
        denuncia.setUsuario(usuario);

        denunciaService.salvarDenuncia(denuncia, foto);

        return "redirect:/";
    }
}
package projeto.projetourbano.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import projeto.model.Denuncia; // Importe a classe Denuncia do pacote correto

@Repository
public interface DenunciaRepository extends JpaRepository<Denuncia, Long> {
}
package projeto.projetourbano.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import projeto.model.Usuario; // Certifique-se que o caminho para a sua entidade Usuario está correto
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<Usuario, Long> {
    Optional<Usuario> findByUsername(String username); // Método para buscar por nome de usuário
}
package projeto.projetourbano.service;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.List;
import java.util.Optional;
import java.util.UUID;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;
import projeto.model.Denuncia;
import projeto.projetourbano.repository.DenunciaRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class DenunciaService {

    private static final Logger logger = LoggerFactory.getLogger(DenunciaService.class);
    private final Path diretorioFotos = Paths.get("uploads");

    @Autowired
    private DenunciaRepository denunciaRepository;

    public Denuncia salvarDenuncia(Denuncia denuncia, MultipartFile foto) {
        denuncia.setStatus("Aberta");

        if (foto != null && !foto.isEmpty()) {
            try {
                Files.createDirectories(diretorioFotos);
                String nomeArquivo = UUID.randomUUID().toString() + "_" + foto.getOriginalFilename();
                Path caminhoArquivo = diretorioFotos.resolve(nomeArquivo);
                Files.copy(foto.getInputStream(), caminhoArquivo);
                denuncia.setCaminhoFoto("/uploads/" + nomeArquivo);
                logger.info("Foto salva com sucesso: {}", caminhoArquivo);
            } catch (IOException e) {
                logger.error("Erro ao salvar foto: {}", e.getMessage());
                e.printStackTrace();
                denuncia.setCaminhoFoto(null);
            }
        } else {
            denuncia.setCaminhoFoto(null);
            logger.warn("Nenhuma foto enviada para a denúncia.");
        }

        return denunciaRepository.save(denuncia);
    }

    public List<Denuncia> listarDenuncias() {
        return denunciaRepository.findAll();
    }

    public Optional<Denuncia> buscarDenunciaPorId(Long id) {
        return denunciaRepository.findById(id);
    }

    public void atualizarStatusDenuncia(Long id, String novoStatus) {
        Optional<Denuncia> denunciaOptional = denunciaRepository.findById(id);
        denunciaOptional.ifPresent(denuncia -> {
            denuncia.setStatus(novoStatus);
            denunciaRepository.save(denuncia);
            logger.info("Status da denúncia {} atualizado para {}", id, novoStatus);
        });
    }
}
package projeto.projetourbano.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import projeto.model.Usuario;
import projeto.projetourbano.repository.UserRepository;
import java.util.Optional;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService implements UserDetailsService {

    private static final Logger logger = LoggerFactory.getLogger(UserService.class);

    @Autowired
    private UserRepository userRepository;

    // Método para buscar um usuário pelo nome de usuário
    public Usuario buscarPorUsername(String username) {
        Optional<Usuario> usuario = userRepository.findByUsername(username);
        return usuario.orElse(null); // Retorna null se não encontrar
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        Optional<Usuario> usuario = userRepository.findByUsername(username);
        if (usuario.isPresent()) {
            Usuario user = usuario.get();
            logger.info("Usuário {} encontrado no banco de dados", username);
            return User.builder() // Use o User.builder() para construir o UserDetails
                    .username(user.getUsername())
                    .password(user.getPassword()) // Assumindo que getPassword() existe
                    .roles("USER") // Defina as roles do usuario
                    .build();
        }
        logger.warn("Usuário {} não encontrado no banco de dados", username);
        throw new UsernameNotFoundException("Usuário não encontrado: " + username);
    }
}
package projeto.projetourbano;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@SpringBootApplication
@EnableJpaRepositories("projeto.projetourbano.repository") // CORREÇÃO AQUI: "repository" em minúsculo
@EntityScan("projeto.model")
public class ProjetourbanoApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProjetourbanoApplication.class, args);
    }

}
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Cadastrar Denúncia Urbana</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-YCJUSJoFGyrPCenYpzNWxlnxcGjaOu/Y0JlPTb9CwHXgswKKFhcgNUYrBAzBYxh" crossorigin="anonymous">
    <style>
        body {
            background-color: #386641; /* Verde escuro */
            padding: 20px;
            color: #f0fff0;
            font-family: sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
        }
        .container {
            max-width: 600px;
            background-color: #f8f9fa; /* Fundo claro para o container */
            padding: 30px;
            border-radius: 8px;
            box-shadow: 0 0 15px rgba(0, 0, 0, 0.2);
        }
        h1 {
            color: #bcbe26; /* Amarelo para o título */
            text-align: center;
            margin-bottom: 30px;
        }
        .form-label {
            color: #bcbe26; /* Branco esverdeado para as labels */
            font-weight: bold;
            margin-bottom: 5px;
            display: block;
        }
        .form-control {
            margin-bottom: 20px;
            border: 1px solid #8FBC8F; /* Borda verde musgo */
            background-color: #e0eee0; /* Verde claro para os campos */
            color: #333;
            padding: 10px;
            border-radius: 4px;
            width: 100%;
            box-sizing: border-box;
        }
        .form-select {
            margin-bottom: 20px;
            border: 1px solid #8FBC8F;
            background-color: #b5cfb5;
            color: #333;
            padding: 10px;
            border-radius: 4px;
            width: 100%;
            box-sizing: border-box;
        }
        .btn-primary {
            background-color: #FFD700; /* Amarelo para o botão */
            border-color: #FFD700;
            color: #386641; /* Texto verde escuro no botão */
            padding: 12px 20px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 1em;
            width: 100%;
        }
        .btn-primary:hover {
            background-color: #E6BE00;
            border-color: #E6BE00;
        }
        .mt-3 {
            margin-top: 20px;
            text-align: center;
        }
        .mt-3 a {
            color: #FFD700;
            text-decoration: none;
        }
        .mt-3 a:hover {
            text-decoration: underline;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Cadastrar Denúncia Urbana</h1>
        <form th:action="@{/denuncias/salvar}" th:object="${denuncia}" method="post" enctype="multipart/form-data">
            <div class="mb-3">
                <label for="categoria" class="form-label">Categoria:</label>
                <select class="form-select" id="categoria" th:field="*{tipo}">
                    <option value="">Selecione a categoria da denúncia</option>
                    <option value="buraco">Buraco na Via</option>
                    <option value="iluminacao">Iluminação Pública Deficiente</option>
                    <option value="lixo">Acúmulo de Lixo</option>
                    <option value="sinalizacao">Sinalização Danificada</option>
                    <option value="vandalismo">Vandalismo</option>
                    <option value="seguranca">Problemas de Segurança</option>
                    <option value="outro">Outro</option>
                </select>
            </div>
            <div class="mb-3">
                <label for="titulo" class="form-label">Título:</label>
                <input type="text" class="form-control" id="titulo" th:field="*{titulo}" required="required">
            </div>
            <div class="mb-3">
                <label for="descricao" class="form-label">Descrição:</label>
                <textarea class="form-control" id="descricao" th:field="*{descricao}" rows="4" required="required"></textarea>
            </div>
            <div class="mb-3">
                <label for="local" class="form-label">Localização (Cidade do DF):</label>
                <select class="form-select" id="local" th:field="*{local}" required="required">
                    <option value="">Selecione a cidade</option>
                    <option value="Brasília">Brasília</option>
                    <option value="Ceilândia">Ceilândia</option>
                    <option value="Samambaia">Samambaia</option>
                    <option value="Planaltina">Planaltina</option>
                    <option value="Taguatinga">Taguatinga</option>
                    <option value="Gama">Gama</option>
                    <option value="Guará">Guará</option>
                    <option value="Santa Maria">Santa Maria</option>
                    <option value="Sobradinho">Sobradinho</option>
                    <option value="Recanto das Emas">Recanto das Emas</option>
                    <option value="Lago Sul">Lago Sul</option>
                    <option value="Riacho Fundo">Riacho Fundo</option>
                    <option value="Paranoá">Paranoá</option>
                    <option value="Núcleo Bandeirante">Núcleo Bandeirante</option>
                    <option value="Cruzeiro">Cruzeiro</option>
                    <option value="São Sebastião">São Sebastião</option>
                    <option value="Candangolândia">Candangolândia</option>
                    <option value="Águas Claras">Águas Claras</option>
                    <option value="Riacho Fundo II">Riacho Fundo II</option>
                    <option value="Sudoeste/Octogonal">Sudoeste/Octogonal</option>
                    <option value="Varjão">Varjão</option>
                    <option value="Park Way">Park Way</option>
                    <option value="Scia">Scia</option>
                    <option value="Estrutural">Estrutural</option>
                    <option value="Sobradinho II">Sobradinho II</option>
                    <option value="Jardim Botânico">Jardim Botânico</option>
                    <option value="Itapoã">Itapoã</option>
                    <option value="Sia">Sia</option>
                    <option value="Vicente Pires">Vicente Pires</option>
                    <option value="Fercal">Fercal</option>
                    <option value="Lago Norte">Lago Norte</option>
                    <option value="Brazlândia">Brazlândia</option>
                </select>
            </div>
            <div class="mb-3">
                <label for="foto" class="form-label">Foto da Denúncia:</label>
                <input type="file" class="form-control" id="foto" name="foto">
                <small class="form-text text-muted">Selecione uma imagem para anexar à denúncia (opcional).</small>
            </div>
            <button type="submit" class="btn btn-primary">Salvar Denúncia</button>
            <p class="mt-3"><a th:href="@{/}">Voltar para a página inicial</a></p>
        </form>
    </div>

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDxyJxEI2PjoAV5czHbbEj9JyHG" crossorigin="anonymous"></script>
</body>
</html>






