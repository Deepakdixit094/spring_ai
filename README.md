package com.spring.rag.config;

import lombok.extern.slf4j.Slf4j;
import org.springframework.ai.document.Document;
import org.springframework.ai.embedding.EmbeddingModel;
import org.springframework.ai.reader.tika.TikaDocumentReader;
import org.springframework.ai.transformer.splitter.TextSplitter;
import org.springframework.ai.transformer.splitter.TokenTextSplitter;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.File;
import java.util.List;

@Slf4j
@Configuration
public class VectorStoreConfig {

    @Bean
    public SimpleVectorStore simpleVectorStore(EmbeddingModel embeddingModel, VectorStoreProperties vectorStoreProperties) {
        SimpleVectorStore store = SimpleVectorStore.builder(embeddingModel).build();

        File vectorStoreFile = new File(vectorStoreProperties.getVectorStorePath());

        if (vectorStoreFile.exists()) {
            store.load(vectorStoreFile);
        } else {
            log.debug("Loading documents into vector store");
            vectorStoreProperties.getDocumentsToLoad().forEach(document -> {
                log.debug("Loading document: " + document.getFilename());
                TikaDocumentReader documentReader = new TikaDocumentReader(document);
                List<Document> docs = documentReader.get();
                TextSplitter textSplitter = new TokenTextSplitter();
                List<Document> splitDocs = textSplitter.apply(docs);
                store.add(splitDocs);
            });

            store.save(vectorStoreFile);
        }

        return store;
    }
}


+++++++++++++
package com.spring.rag.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;

import java.util.List;


@Configuration
@ConfigurationProperties(prefix = "sfg.aiapp")
public class VectorStoreProperties {

    private String vectorStorePath;
    private List<Resource> documentsToLoad;

    public String getVectorStorePath() {
        return vectorStorePath;
    }

    public void setVectorStorePath(String vectorStorePath) {
        this.vectorStorePath = vectorStorePath;
    }

    public List<Resource> getDocumentsToLoad() {
        return documentsToLoad;
    }

    public void setDocumentsToLoad(List<Resource> documentsToLoad) {
        this.documentsToLoad = documentsToLoad;
    }
}

+++++++++++++++++

package com.spring.rag.controller;

import com.spring.rag.model.Answer;
import com.spring.rag.model.Question;
import com.spring.rag.services.RagService;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;


@RestController
public class RagController {

    private final RagService ragService;

    public RagController(RagService ragService) {
        this.ragService = ragService;
    }

    @PostMapping("/ask")
    public Answer askQuestion(@RequestBody Question question) {
        return ragService.getAnswer(question);
    }


}


++++++++++++++++
package com.spring.rag.model;

import lombok.*;

@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@ToString
public class Answer {

    private String answer;

}


++++++++++++++++
package com.spring.rag.model;

import lombok.*;

@AllArgsConstructor
@NoArgsConstructor
@Getter
@Setter
@ToString
public class Question {

    private String question;
}


++++++++++++++++++++
package com.spring.rag.services;

import com.spring.rag.model.Answer;
import com.spring.rag.model.Question;
import org.springframework.stereotype.Component;

@Component
public interface RagService {


    Answer getAnswer(Question question);


}

++++++++++++++++++++++++
package com.spring.rag.services;

import com.spring.rag.model.Answer;
import com.spring.rag.model.Question;
import lombok.RequiredArgsConstructor;
import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.chat.prompt.PromptTemplate;
import org.springframework.ai.document.Document;
import org.springframework.ai.ollama.OllamaChatModel;
import org.springframework.ai.vectorstore.SearchRequest;
import org.springframework.ai.vectorstore.SimpleVectorStore;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@RequiredArgsConstructor
@Service
public class RagServiceImpl implements RagService {

    private final OllamaChatModel chatModel;
    private final SimpleVectorStore vectorStore;

    @Value("classpath:/templates/rag-prompt-template.st")
    private Resource ragPromptTemplate;

    @Override
    public Answer getAnswer(Question question) {
        List<Document> documents = vectorStore.similaritySearch(SearchRequest.builder()
                .query(question.getQuestion())
                .topK(2)
                .build());

        List<String> contentList = documents.stream().map(Document::getContent).toList();

        PromptTemplate promptTemplate = new PromptTemplate(ragPromptTemplate);
        Prompt prompt = promptTemplate.create(Map.of("input", question.getQuestion(), "documents", String.join("\n", contentList)));

        contentList.forEach(System.out::println);

        ChatResponse response = chatModel.call(prompt);

        return new Answer(response.getResult().getOutput().getContent());
    }
}

++++++++++++++++++++
You are a helpful assistant, conversing with a user about the subjects contained in a set of documents.
Use the information from the DOCUMENTS section to provide accurate answers. If unsure or if the answer
isn't found in the DOCUMENTS section, simply state that you don't know the answer.Keep the answer short and crisp.


QUESTION:
{input}

DOCUMENTS:
{documents}


+++++++++++++++++++
You are a helpful assistant, conversing with a user about the subjects contained in a set of documents.
Use the information from the DOCUMENTS section to provide accurate answers. If unsure or if the answer
isn't found in the DOCUMENTS section, simply state that you don't know the answer.

QUESTION:
{input}

The documents are in a tabular dataset containing the following columns:
id,title (title of move),genres,original_language,overview,production_companies,release_date,budget (USD, format as $0,000),revenue (USD, format as $0,000),runtime (in minutes - display as hours and minutes),credits (cast or actors)

DOCUMENTS:
{documents}

++++++++++++++++++++++++

spring:
  application:
    name: springrag
  ai:
    ollama:
      chat:
        options:
          model: deepseek-r1:8b
      embedding:
        enabled: true
        options:
          model: all-minilm:33m

sfg:
  aiapp:
    vectorStorePath: E:/Temp/vectorstore.json
    documentsToLoad:
      - classpath:/movies500Trimmed.csv

